# fleetscan design

Date: 2026-09-17
Status: approved, not yet implemented

## Problem

An operator who administers many Azure tenants from one workstation cannot cheaply answer a
fleet-wide question. "Which of my AKS clusters run a Kubernetes version Azure no longer supports,
and what can each one upgrade to?" requires enumerating every tenant, every subscription and every
cluster, then comparing each cluster against the version list Azure offers in that cluster's
region. Done by hand through the Azure CLI that is one process invocation per tenant per query,
run serially, and the result is terminal scrollback rather than a report.

fleetscan answers the question in one run.

## Scope

v1 is one command performing one check:

1. enumerate every context in the local cloudctx registry
2. for each context, enumerate every subscription the caller can see
3. for each subscription, list AKS clusters and compare each cluster's Kubernetes version against
   the versions Azure currently offers in that cluster's region
4. render a table, JSON or CSV, and exit non-zero when anything is out of support

### Non-goals for v1

- AWS contexts. cloudctx supports them; fleetscan does not.
- Any write. fleetscan never mutates a tenant and never writes into `AZURE_CONFIG_DIR` or the AWS
  credential files.
- Further checks. The probe interface exists because it is the natural shape of the unit, not
  because a plugin system is planned. One probe ships.
- Caching, persistence, dashboards, scheduled runs.

## The constraint that shapes the design

cloudctx isolates tenants by giving each context its own `AZURE_CONFIG_DIR`, a directory holding
the Azure CLI's MSAL token cache and its active subscription. Isolation is carried entirely by one
environment variable.

In Go the process environment is global. `os.Setenv` cannot be scoped to a goroutine. A design that
set `AZURE_CONFIG_DIR` per tenant and scanned tenants concurrently in one process would have
goroutines overwriting each other's credentials mid-flight, which is exactly the failure cloudctx
exists to prevent. The design must make that impossible structurally, not avoid it carefully.

Therefore every credential enters the process through a child process, and nothing else in
fleetscan touches a credential store. Once per context, fleetscan runs:

    cloudctx exec <name> -- az account get-access-token \
        --resource https://management.azure.com --output json

and uses the returned bearer token in-process for every subsequent call. That is one child process
per context in total, not one per query.

## Architecture

| Unit | Responsibility | Knows about |
|---|---|---|
| `internal/ctxsource` | Turn the local cloudctx registry into `[]Context` | cloudctx, `os/exec` |
| `internal/auth` | Turn a `Context` into an `azcore.TokenCredential` | the Azure CLI, `os/exec` |
| `internal/probe` | The check abstraction and its one implementation | the Azure SDK |
| `internal/report` | Render findings, decide the exit code | nothing external |
| `cmd/fleetscan` | Flags, wiring, fan-out | all of the above |

Only `ctxsource` knows cloudctx exists. Only `auth` knows a credential ever came from a CLI. Probes
receive an `azcore.TokenCredential` and cannot tell how it was obtained.

## Interfaces

```go
// internal/ctxsource
type Context struct {
    Name     string // cloudctx context name
    TenantID string // azure_tenant, from `cloudctx show <name>`
    Store    string // the store: path, from `cloudctx show <name>`
}

type Source interface {
    Contexts(ctx context.Context) ([]Context, error)
}

// Runner is the single seam through which any unit reaches a child process.
// Tests supply a fake; nothing else in the package spawns anything.
type Runner interface {
    Run(ctx context.Context, name string, args ...string) ([]byte, error)
}
```

```go
// internal/auth
// Credential satisfies azcore.TokenCredential. The Azure SDK requires no
// declaration that it does; having the method is enough.
type Credential struct { /* context name, runner, cached token, expiry, mutex */ }

func (c *Credential) GetToken(
    ctx context.Context, opts policy.TokenRequestOptions,
) (azcore.AccessToken, error)
```

```go
// internal/probe
type Subscription struct {
    Context      ctxsource.Context
    ID           string
    DisplayName  string
}

type Status string

const (
    StatusOK          Status = "ok"          // supported, latest patch
    StatusOutdated    Status = "outdated"    // supported minor, newer patch exists
    StatusUnsupported Status = "unsupported" // minor no longer offered in region
    StatusUnavailable Status = "unavailable" // the context could not be scanned
)

type Finding struct {
    Context      string
    Subscription string
    Resource     string
    Location     string
    Status       Status
    Current      string
    Available    []string
    Detail       string
}

type Probe interface {
    Name() string
    Run(ctx context.Context, sub Subscription, cred azcore.TokenCredential) ([]Finding, error)
}
```

## Data flow

    cloudctx list --names
      -> []Context                                     (ctxsource)
           -> auth.Credential per context              (one az child process each)
                -> armsubscriptions.List               -> []Subscription
                     -> probe.Run per subscription     -> []Finding
                          -> report                    -> table | --json | --csv, exit code

The outer fan-out is over contexts, the inner over subscriptions.

## Concurrency

`errgroup.WithContext` at the context level behind a counting semaphore, `--parallel`, default 8.
A second `errgroup` fans out over the subscriptions within one context. `context.Context` carries
cancellation from a `--timeout` and from SIGINT, so an interrupted run stops spawning work rather
than orphaning child processes.

Per-context failures are collected rather than propagated. The outer `errgroup` therefore returns
an error only for a fault that invalidates the whole run, such as cloudctx being absent or too old.

## Error policy

A context that cannot be authenticated produces one `Finding` with `StatusUnavailable` and the
reason, instead of aborting the run.

The rationale is that privileged tenant access is commonly time-boxed through just-in-time
elevation, so an unelevated or expired tenant is routine rather than exceptional. A run that
aborted on the first such tenant would be useless in practice. A run that silently omitted it would
be worse than useless, because the operator would read a clean table and conclude the fleet is
healthy. The failure has to be visible in the output.

Exit codes:

| Code | Meaning |
|---|---|
| 0 | every context reachable, every cluster supported |
| 1 | at least one cluster out of support |
| 2 | at least one context unreachable, and no cluster out of support |
| 3 | usage or configuration error, including cloudctx missing or older than 1.4.0 |

Code 1 takes precedence over code 2.

## Version policy

Supported versions are whatever Azure reports for the cluster's region, fetched through the
container service client. No support window is hardcoded, so there is no policy to maintain as
Azure's schedule moves.

Classification, given a cluster's current version and the versions offered in its region:

- `unsupported`: the cluster's minor version is not offered in that region
- `outdated`: the minor is offered, but a higher patch exists within it
- `ok`: otherwise

## Output

Table by default. `--json` and `--csv` for feeding a report. `--context <name>` narrows the run to
one or more contexts, `--parallel` sets the semaphore width, `--timeout` bounds the whole run.

## Testing

- `ctxsource` and `auth` take a `Runner`, so their tests run against recorded `cloudctx` and `az`
  output with no process spawned. Fixtures use synthetic GUIDs and placeholder names.
- The classification logic is a pure function, `classify(current string, offered []string)
  (Status, []string)`, table-driven tested with no IO at all. The only genuinely interesting logic
  in the tool needs neither a cloud nor a network to test.
- `report` rendering is golden-file tested.
- No test requires network access or an Azure account.

## Dependencies

- `github.com/Azure/azure-sdk-for-go/sdk/azcore`
- `github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/containerservice/armcontainerservice`
- `github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/resources/armsubscriptions`
- `golang.org/x/sync/errgroup`

Flags use the standard library `flag` package. One command does not justify a CLI framework.

## Compatibility

cloudctx 1.4.0 or newer. fleetscan uses only surfaces published in the cloudctx companion contract:

- `cloudctx --version`, for the compatibility gate
- `cloudctx list --names`
- `cloudctx show <name>`, reading `azure_tenant` and the `store:` path
- `cloudctx exec <name> -- ...`

No internal, underscore-prefixed cloudctx command is used. Companion state, should fleetscan ever
need any, belongs under `$CLOUDCTX_STORE/fleetscan/`.
