# fleetscan design

Date: 2026-09-17
Status: approved, not yet implemented

## Problem

An operator administering many Azure tenants from one workstation cannot cheaply answer a fleet-wide
question. "Which of my AKS clusters run a Kubernetes version Azure no longer supports, which are
unhealthy now, and which will refuse to upgrade?" means enumerating every tenant, subscription and
cluster, then comparing each against what Azure offers in its region. By hand through the Azure CLI
that is one process invocation per tenant per query, run serially, and the result is terminal
scrollback rather than a report. Asking the same about one suspect cluster is a different problem:
there nobody wants a fleet table, they want everything the platform knows about that cluster,
including Azure's own diagnostics. fleetscan answers both, from one binary, at two zoom levels.

## Scope

| Command | Breadth | Depth | Cost per subscription |
|---|---|---|---|
| `fleetscan scan` | every registered context | cheap checks only | a fixed handful of calls |
| `fleetscan doctor <cluster>` | one cluster | every check plus detectors | one cluster's worth |

Both share context discovery, authentication, the probe interface and reporting. `doctor` is a
selection and verbosity difference, not a second codebase: it runs the same probes with `ModeDoctor`
in the selection, adds the detector probes, and prints per-resource detail instead of one row per
cluster. fleetscan is read-only in both modes: it never mutates a tenant, and never writes into
`AZURE_CONFIG_DIR` or any credential file.

## The constraint that shapes the design

cloudctx isolates tenants by giving each context its own `AZURE_CONFIG_DIR`, a directory holding the
Azure CLI's token cache and its active subscription. Isolation is carried entirely by one environment
variable.

In Go the process environment is global, and `os.Setenv` cannot be scoped to a goroutine. A design
setting `AZURE_CONFIG_DIR` per tenant and scanning concurrently in one process would have goroutines
overwriting each other's credentials mid-flight, exactly the failure cloudctx exists to prevent. That
has to be impossible structurally, not avoided carefully, so every credential enters the process
through a child process and nothing else in fleetscan touches a credential store. Once per context:

    cloudctx exec <name> -- az account get-access-token \
        --resource https://management.azure.com --output json

and uses that bearer token in-process for everything after: one child process per context in total,
not one per query. The token is wrapped in a hand-written type satisfying `azcore.TokenCredential`, so
the rest of the program sees an ordinary Azure SDK credential.

## Architecture

Units, each with one job. Only `ctxsource` knows cloudctx exists, only `auth` knows a credential came
from a CLI, and probes get an `azcore.TokenCredential` and cannot tell how it was obtained. That last
property is what lets the detector client reuse the credential with no new auth work.

| Unit | Responsibility |
|---|---|
| `internal/ctxsource` | Turn the local cloudctx registry into `[]Context{Name, TenantID, Store}` |
| `internal/auth` | Turn a `Context` into an `azcore.TokenCredential`, refreshing on expiry |
| `internal/probe` | The `Probe` interface and its implementations |
| `internal/detector` | Hand-rolled ARM client for AKS detectors |
| `internal/report` | Render findings as table, JSON or CSV; decide the exit code |
| `cmd/fleetscan` | Flags, subcommand wiring, fan-out |

## Interfaces

```go
// internal/ctxsource
type Context struct {
    Name     string // cloudctx context name
    TenantID string // azure_tenant, from `cloudctx show <name>`
    Store    string // the store: path, from `cloudctx show <name>`
}

// Runner is the single seam through which any unit reaches a child process.
// Tests supply a fake; nothing else in the package spawns anything.
type Runner interface {
    Run(ctx context.Context, name string, args ...string) ([]byte, error)
}

// internal/probe
type Status string

const (
    StatusOK          Status = "ok"
    StatusOutdated    Status = "outdated"    // supported, but a newer patch exists
    StatusUnsupported Status = "unsupported" // outside the support window
    StatusPlatform    Status = "platform"    // N-3: platform support only, no SLA or patches
    StatusDegraded    Status = "degraded"    // Resource Health or a detector reports a problem
    StatusUnavailable Status = "unavailable" // the context or subscription could not be scanned
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
    Mode() Mode // ModeScan, ModeDoctor, or ModeBoth
    Run(ctx context.Context, sub Subscription, cred azcore.TokenCredential) ([]Finding, error)
}
```

Supporting types, kept out of that block so it stays the contract. `Mode` is a `uint8` bit set with
`ModeScan`, `ModeDoctor` and `ModeBoth`. `Subscription` is a `ctxsource.Context` plus the subscription
ID and display name. `ctxsource.Source` returns `[]Context`. `auth.Credential` holds a context name, a
`Runner`, a cached token, an expiry and a mutex, and satisfies `azcore.TokenCredential` just by having
`GetToken(context.Context, policy.TokenRequestOptions) (azcore.AccessToken, error)`.

The pipeline:

    cloudctx list --names
      -> []Context                                     (ctxsource)
           -> auth.Credential per context              (one az child process each)
                -> armsubscriptions.List               -> []Subscription
                     -> probe.Run per subscription     -> []Finding
                          -> report                    -> table | --json | --csv, exit code

The outer fan-out is over contexts, the inner over subscriptions; `doctor` walks the same path and
stops at the one subscription and cluster it was asked about.

## Scan mode

**The fan-in rule: fleet-wide checks must use APIs whose cost scales per subscription, not per
cluster.** A check issuing one call per cluster is fine against three clusters and produces
`SubscriptionRequestsThrottled` (HTTP 429) across a large fleet, leaving the run slow and incomplete.
Every scan-mode source is subscription-scoped for that reason, and the one inherently per-cluster
family of calls, the detectors, is therefore a `doctor` feature rather than something rate-limited
into `scan`.

| Source | Call | Cost |
|---|---|---|
| AKS inventory | `managedClusters` LIST, 2025-09-01, follow `nextLink` | 1 per subscription |
| Resource Health | `availabilityStatuses` `listBySubscriptionId`, 2025-05-01 | 1 per subscription |
| Advisor | recommendations, filtered by category and severity | 1 per subscription |
| Offered versions | `ListKubernetesVersions` | 1 per location per run |

Resource Health is `Microsoft.ResourceHealth/availabilityStatuses`. It supports `$filter` and
`$expand=recommendedactions`, and one call per subscription covers every cluster in it. Statuses are
Available, Unavailable, Degraded and Unknown, where Unknown means no data for 10 or more minutes.
fleetscan filters server side to `Microsoft.ContainerService/managedClusters` and maps Unavailable,
Degraded and Unknown to `StatusDegraded`, carrying the reason into `Detail`. Unknown counts, because a
cluster the platform has lost sight of is not a cluster known to be healthy.

Advisor recommendations are filterable by category (Cost, HighAvailability, Performance, Security) and
by severity, and are subscription-scoped too. Scan pulls HighAvailability and Security, doctor all
four; Advisor findings annotate and never set an exit code alone. `ListKubernetesVersions` is fetched
once per distinct location per run and held for that run only, with no on-disk cache.

### Fields read from managedClusters

- `provisioningState` (Succeeded, Failed, Canceled, or a non-terminal value meaning an operation is in
  flight) and `powerState.code`.
- `currentKubernetesVersion`; `sku.tier` and `supportPlan` (`KubernetesOfficial` or
  `AKSLongTermSupport`).
- `agentPoolProfiles[]`: per pool `provisioningState`, `powerState`, `orchestratorVersion`,
  `nodeImageVersion`, `count`, `minCount`, `maxCount`.
- `autoUpgradeProfile.upgradeChannel` and `.nodeOSUpgradeChannel`.
- `networkProfile.outboundType`, `loadBalancerProfile.allocatedOutboundPorts`, outbound IP count.
- `apiServerAccessProfile` (private cluster, authorized IP ranges), `disableLocalAccounts`,
  `aadProfile.enableAzureRBAC` and `nodeResourceGroup`.

**Footgun: if `powerState` is missing or null, treat it as unknown. Do not infer it from
`provisioningState`.** A cluster can be Succeeded and Stopped at once, and a stopped cluster a report
calls healthy is a worse answer than one it calls unknown.

### Two checks that are arithmetic, not calls

Both run on data the inventory call already returned, so they cost nothing extra, and both are pure
functions. That matters twice: they are the most testable code in the project, table-driven with no
IO, and they predict failures that otherwise surface mid-upgrade with a pool already in `Failed`.

- **SNAT headroom.** Available ports = outbound IP count x 64,000; required = node count x
  `allocatedOutboundPorts`. That inequality is exactly the
  `InvalidLoadBalancerProfileAllocatedOutboundPorts` error, checkable before it fires, not after.
- **Node pool version skew.** A node pool's minor version may not be more than 3 behind the control
  plane. Violating it is `NodePoolMcVersionIncompatible`, and the arithmetic is two integers.

## Version classification policy

The policy behind one pure function,
`classify(current string, offered []string, tier string, plan string) (Status, []string)`:

- N, N-1 and N-2 are community support.
- N-3 is **platform support only**: Azure platform issues are supported, but there is no AKS SLA, no
  support for Kubernetes components, no security patches and no new cluster creation on that version.
  Calling that plain "unsupported" loses the distinction that matters to whoever schedules the
  upgrade, so it gets its own status, `StatusPlatform`.
- When a new minor version reaches GA, the oldest supported minor drops out of support 30 days later,
  so a cluster can change status without anything about the cluster changing.
- Long Term Support requires the Premium tier plus `supportPlan = AKSLongTermSupport`, and covers only
  the two most recent patches of that minor version.
- So `sku.tier` and `supportPlan` change the answer: an LTS cluster on Premium can be supported where
  an identical Free-tier cluster is not. The classification cannot be written from the version string
  alone, which is why tier and plan are parameters, not context the caller keeps to itself.
- Upgrades must be sequential through minor versions, except when climbing out of an unsupported
  version, where a jump is permitted. The path in `Available` follows that rule, so what fleetscan
  prints is a path Azure will accept. No support window is hardcoded as dates: the offered list comes
  from Azure per region, and this policy is applied to it.

## Doctor mode: the detectors client

**The load-bearing detail: AKS detectors are an ARM pass-through to AppLens. They are not in the AKS
swagger, so no typed Go SDK client exists for them.** fleetscan hand-rolls two GETs with a raw
`http.Client`, using the same `azcore.TokenCredential` already built for everything else
(`policy.TokenRequestOptions{Scopes: []string{"https://management.azure.com/.default"}}`), so adding
detectors costs no new authentication work at all.

```
List:  GET https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}
       /providers/Microsoft.ContainerService/managedClusters/{cluster}
       /detectors?api-version=2024-08-01

Run:   GET .../managedClusters/{cluster}/detectors/{detectorName}
       ?startTime={rfc3339}&endTime={rfc3339}&api-version=2024-08-01
```

- RBAC: `Microsoft.ContainerService/managedClusters/detectors/read`, so Reader is enough. Both calls
  are GETs, which keeps detectors inside the never-writes promise.
- List response: `value[]` of `{id, name, type, location, properties{metadata{id, name, category,
  description, type}, status{message, statusId}}}`. `statusId` is the numeric severity, and it is the
  field worth surfacing.
- Run response: `properties{dataset[]{renderingProperties{title, description, isVisible, type},
  table{tableName, columns[{columnName, dataType}], rows[][]}}, metadata, status}`. Rows are untyped,
  so any check reading a specific detector must pin its columns and fail loudly if the shape changes.
- Time constraints, enforced server side and mirrored client side so a bad window fails locally:
  RFC3339, both ends within the last 30 days, `end > start`, window at most 24 hours.
- The eight categories, exact strings, compared case-insensitively: `Best Practices`,
  `Cluster and Control Plane Availability and Performance`, `Connectivity Issues`,
  `Create, Upgrade, Delete and Scale`, `Deprecations`, `Identity and Security`, `Node Health`,
  `Storage`.
- There is **no server-side category filter**. List, filter client side on
  `properties.metadata.category`, then run each match by `properties.metadata.id`.
- **Detector names are dynamic per cluster. Always list first; never hard-code a name.**
- Cache the detector list per `subscription:rg:cluster` for the run, and honor `Retry-After` on 429.

## Why doctor mode needs no kubectl

There are two layers an AKS cluster can be inspected from. The ARM control plane is always reachable
with the token fleetscan already holds: it works on private clusters and needs no kubeconfig, no
cluster RBAC and no network path to the Kubernetes API server. The data plane needs all three, and
from a workstation a private cluster is unreachable without a jump host or a VPN. Detectors make
ARM-only sufficient rather than a compromise: they return the platform's own analysis of node health,
connectivity and storage without the caller touching the data plane, so the deep mode reports Azure's
diagnosis instead of fleetscan's guess.

Out of reach, listed so nobody is surprised by its absence: node conditions and kubelet pressure
detail, Pending or FailedScheduling pods, OOMKills, Pod Disruption Budgets blocking a drain, service
endpoint and selector mismatches, CoreDNS health.

Operational note, printed alongside any node finding: AKS node auto-repair reboots, then reimages,
then redeploys any node NotReady for more than five minutes, emitting events from source
`aks-auto-repair` (`NodeRebootStart`, `NodeReimageStart`, `NodeRedeployStart`). A NotReady node may be
mid-remediation rather than stuck, so the report says so rather than implying somebody must act.

## Pre-upgrade readiness checklist

AKS runs seven validations before an upgrade. Every one is ARM-checkable in advance, which makes them
a ready-made set of checks rather than a list somebody had to invent.

| Validation | v1 | Why |
|---|---|---|
| Valid upgrade path | scan | Arithmetic on the offered version list |
| Quota for surge nodes | scan | Subscription usage API, subscription-scoped |
| Subnet IP headroom | scan | Subnet address count against the surge requirement |
| Resource lock on the `MC_` group | scan | One lock query |
| API breaking changes (deprecated API usage) | doctor | Needs a detector run |
| Pod Disruption Budget configuration | doctor | Data plane, reachable only through a detector |
| Expired certificates or service principals | doctor | Needs a detector run |

Deprecated APIs: AKS blocks a minor-version control plane upgrade if deprecated API usage was seen in
the 12 hours before the attempt. It targets 1.26 and later, excludes the read-only Get, List and Watch
verbs, records usage hourly, and surfaces through the `Kubernetes API deprecations` detector under the
`Create, Upgrade, Delete and Scale` category.

Certificates: only a full control plane plus node pool upgrade renews expired cluster certificates, so
"upgrade the node image" is the wrong advice for an expired certificate and fleetscan does not give
it. A node image upgrade or a same-version node pool upgrade renews nothing.

## Concurrency, errors and exit codes

`errgroup.WithContext` at the context level behind a counting semaphore (`--parallel`, default 8), and
a second `errgroup` over the subscriptions within one context. `context.Context` carries cancellation
from `--timeout` and SIGINT, so an interrupted run stops spawning work rather than orphaning child
processes. Per-context failures are collected, not propagated: the outer group returns an error only
for a fault invalidating the whole run, such as cloudctx being absent or older than 1.4.0.

A context that cannot authenticate produces one `Finding` with `StatusUnavailable` and the reason.
Privileged tenant access is commonly time-boxed through just-in-time elevation, so an unelevated or
expired tenant is routine, not exceptional. A run aborting on the first one would be useless, and one
silently omitting it worse than useless, because a clean table reads as a healthy fleet. The failure
has to be visible in the output.

Throttling: honor `Retry-After`, then bounded exponential backoff. The per-subscription fan-in rule is
the primary defence against 429; retry logic is the fallback for what it misses, not the strategy.

| Code | Meaning |
|---|---|
| 0 | every context reachable, nothing out of support, nothing degraded |
| 1 | at least one cluster unsupported, degraded, or failing a readiness check |
| 2 | at least one context or subscription unreachable, and nothing in code 1 |
| 3 | usage or configuration error, including cloudctx missing or older than 1.4.0 |

Precedence: 1 beats 2 beats 0.

## Output

`scan` prints a table by default, with `--json` and `--csv` for feeding a report. `--context` is
repeatable and narrows the run, `--parallel` sets the semaphore width, `--timeout` bounds the run.

```
$ fleetscan scan
CONTEXT      SUBSCRIPTION     CLUSTER       LOCATION     VERSION  STATUS        DETAIL
acme-prod    platform         aks-prod-01   westeurope   1.30.4   outdated      1.30.7 offered
acme-prod    platform         aks-prod-02   westeurope   1.28.9   platform      N-3, no patches
acme-prod    platform         aks-prod-03   westeurope   1.31.3   degraded      SNAT 120k > 64k
beta-corp    shared-services  aks-shared    northeurope  1.31.3   ok
beta-corp    shared-services  aks-batch     northeurope  1.27.6   unsupported   upgrade to 1.29.x
gamma-ab     -                -             -            -        unavailable   no valid token

6 clusters, 4 need attention, 1 context unreachable
$ echo $?
1
```

`doctor <cluster>` prints full detail for one cluster, in the same output formats. `--category`
narrows the detectors and `--since` sets the window (default 24 hours, maximum 24 hours, inside the
last 30 days). If the cluster name matches in more than one context, doctor exits 3 and lists the
matches; `--context` disambiguates.

```
$ fleetscan doctor aks-prod-01 --context acme-prod
cluster    aks-prod-01  westeurope  sub platform (00000000-0000-0000-0000-000000000000)
groups     rg-platform-prod, node group MC_rg-platform-prod_aks-prod-01_westeurope
plane      1.30.4 outdated (1.30.7 offered), Standard tier, supportPlan KubernetesOfficial
state      provisioning Succeeded, power Running, resource health Available, channels patch/NodeImage
access     private cluster, 2 authorized ranges, Azure RBAC on

pools      system  3 nodes (3/5)   1.30.4  Succeeded  skew ok
           user    15 nodes (3/30) 1.29.8  Succeeded  skew ok

readiness  upgrade path ok (1.30.4 -> 1.30.7), surge quota ok (33 of 100 vCPU), MC_ lock ok
           subnet headroom  warn  41 free addresses, 15 needed at 33 percent surge
           SNAT headroom    fail  1 IP = 64000 ports, 15 nodes x 8000 = 120000 needed

detectors  window 2026-09-16T08:00:00Z to 2026-09-17T08:00:00Z, statusId after the name
           Node Health / node-health-summary      2  1 node NotReady for 3 minutes
           Connectivity Issues / cluster-dns      0  no issues found
           Create, Upgrade... / api-deprecations  1  2 deprecated API calls in 12 hours

note: a node NotReady under five minutes may be mid auto-repair, not stuck
$ echo $?
1
```

## Testing

- `ctxsource` and `auth` take a `Runner`, so their tests run against recorded `cloudctx` and `az`
  output with no process spawned. Fixtures use synthetic GUIDs and placeholder names.
- The `detector` client is tested against an `httptest.Server` serving recorded bodies, including a
  429 with `Retry-After` and a malformed table. No live ARM call in any test.
- `classify()` and the SNAT and version-skew arithmetic are pure functions, table-driven tested, with
  no IO. The genuinely interesting logic here needs neither a cloud nor a network.
- `report` rendering is golden-file tested for table, JSON and CSV. No test requires network access or
  an Azure account.

## Dependencies and compatibility

`github.com/Azure/azure-sdk-for-go/sdk/azcore`; under
`github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/`, the packages
`containerservice/armcontainerservice`, `resources/armsubscriptions`,
`resourcehealth/armresourcehealth` and `advisor/armadvisor`; and `golang.org/x/sync/errgroup`.
Detectors use the standard library `net/http`, because there is no generated client to use instead,
and flags use the standard library `flag` package: two subcommands do not justify a CLI framework.

cloudctx 1.4.0 or newer. fleetscan uses only surfaces published in the cloudctx companion contract:
`cloudctx --version` for the gate, `cloudctx list --names`, `cloudctx show <name>` (reading
`azure_tenant` and the `store:` path), and `cloudctx exec <name> -- ...`. No internal,
underscore-prefixed cloudctx command is used. Companion state, should fleetscan ever need any, belongs
under `$CLOUDCTX_STORE/fleetscan/`.

## Implementation ownership

This project doubles as a Go learning exercise for its author, so the split is recorded here.

| Part | Owner |
|---|---|
| `internal/ctxsource` | Written first, as a worked reference to read before starting |
| `auth.GetToken`, satisfying `azcore.TokenCredential` | The author |
| `classify()`, SNAT and version-skew arithmetic | The author |
| Concurrency and error policy in `cmd/fleetscan` | The author |
| Scaffolding, `internal/detector`, `internal/report`, CI | Shared |

## Out of scope for v1

- AWS contexts. cloudctx supports them; fleetscan does not.
- Any write path of any kind.
- The Kubernetes data plane, and any dependency on kubectl or a kubeconfig.
- Caching beyond the per-run detector list and the per-run offered-version list.
- Dashboards, persistence, scheduled runs.
