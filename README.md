# fleetscan

Read-only fleet checks across every Azure tenant on your workstation, in one run.

fleetscan enumerates the contexts registered with
[cloudctx](https://github.com/eliknut/cloudctx), obtains one access token per context through
cloudctx's isolated credential store, and then scans every subscription concurrently. It has two
subcommands over the same engine, at two zoom levels.

## `fleetscan scan`

Fleet-wide, across every registered context, using only checks whose cost scales per subscription
rather than per cluster. It reports Kubernetes versions against what Azure currently offers in each
cluster's region (including the platform-support-only N-3 band), cluster and node pool state, Azure
Resource Health, SNAT port headroom, node pool version skew, and the pre-upgrade validations that can
be answered from the control plane.

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

An unreachable tenant is reported as a row rather than swallowed or treated as fatal, because
time-boxed elevation makes an unelevated tenant routine and a clean table would otherwise read as a
healthy fleet.

## `fleetscan doctor <cluster>`

One cluster, in depth. Everything `scan` knows about it, plus full per-resource detail and the AKS
detectors: Azure's own diagnostics for node health, connectivity, storage, deprecated API usage and
upgrade blockers. Detectors are per-cluster calls, which is why they live here and not in the
fleet-wide scan.

Doctor mode needs no kubectl and no kubeconfig. It reads everything through the ARM control plane
with the token it already holds, so it works against private clusters with no jump host and no VPN.

```
$ fleetscan doctor aks-prod-01 --context acme-prod
plane      1.30.4 outdated (1.30.7 offered), Standard tier, supportPlan KubernetesOfficial
state      provisioning Succeeded, power Running, resource health Available
readiness  upgrade path ok, surge quota ok, subnet headroom warn, SNAT headroom fail
detectors  Node Health / node-health-summary   2  1 node NotReady for 3 minutes
           Connectivity Issues / cluster-dns   0  no issues found
```

Output is a table by default, with `--json` and `--csv` in both modes.

## Why a separate tool

The Azure CLI keeps its token cache and active subscription in one directory, `AZURE_CONFIG_DIR`.
cloudctx gives each context its own, which is what makes several tenants safe to work with from one
machine. That variable is per process, so a concurrent scanner cannot simply switch it between
tenants. fleetscan takes one token per context through a child process and does everything else
in-process, which keeps cloudctx's isolation intact while still scanning tenants in parallel.

## Status

Design complete, implementation not started. See
[docs/design](docs/design/2026-09-17-fleetscan-design.md).

## Requirements

- Go 1.24 or newer
- cloudctx 1.4.0 or newer
- the Azure CLI, logged in to the contexts you want scanned

## Safety

fleetscan never writes. It does not mutate any Azure resource, and it never writes into
`AZURE_CONFIG_DIR` or the AWS credential files. Every call it makes, detectors included, is a GET.
Access is whatever the signed-in identity already has; the tool requests nothing further.

## Licence

MIT
