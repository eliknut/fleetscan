# fleetscan

Read-only fleet checks across every Azure tenant on your workstation, in one run.

fleetscan enumerates the contexts registered with
[cloudctx](https://github.com/eliknut/cloudctx), obtains one access token per context through
cloudctx's isolated credential store, and then scans every subscription concurrently. v1 ships one
check: which AKS clusters run a Kubernetes version Azure no longer offers in their region, and what
each one can upgrade to.

```
$ fleetscan
CONTEXT      SUBSCRIPTION     CLUSTER          LOCATION     VERSION   STATUS        AVAILABLE
acme-prod    platform         aks-prod-01      westeurope   1.30.4    outdated      1.30.7
acme-prod    platform         aks-prod-02      westeurope   1.28.9    unsupported   1.30.7, 1.31.3
beta-corp    shared-services  aks-shared       northeurope  1.31.3    ok
gamma-ab     -                -                -            -         unavailable   no valid token

2 clusters need attention, 1 context unreachable
$ echo $?
1
```

## Why a separate tool

The Azure CLI keeps its token cache and active subscription in one directory, `AZURE_CONFIG_DIR`.
cloudctx gives each context its own, which is what makes several tenants safe to work with from one
machine. That variable is per process, so a concurrent scanner cannot simply switch it between
tenants. fleetscan takes one token per context through a child process and does everything else
in-process, which keeps cloudctx's isolation intact while still scanning tenants in parallel.

## Status

Design complete, implementation not started. See
[docs/superpowers/specs](docs/superpowers/specs/2026-09-17-fleetscan-design.md).

## Requirements

- Go 1.24 or newer
- cloudctx 1.4.0 or newer
- the Azure CLI, logged in to the contexts you want scanned

## Safety

fleetscan never writes. It does not mutate any Azure resource, and it never writes into
`AZURE_CONFIG_DIR` or the AWS credential files. Access is whatever the signed-in identity already
has; the tool requests nothing further.

## Licence

MIT
