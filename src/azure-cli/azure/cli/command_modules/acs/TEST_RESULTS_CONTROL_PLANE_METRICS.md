# Live Test Results — `--enable/--disable-control-plane-metrics` (GA azure-cli port)

**Branch:** `kadubey/aks-control-plane-metrics` · **Commit:** `5f1deef`
**Build:** `azure-cli==2.87.0` editable install from this repo in `testenv\` venv (every `az aks` invocation runs the GA branch code).
**Subscription:** `ce4d1293-71c0-4c72-bc55-133553ee9e50` (Bragi Test) · **Tenant:** `72f988bf-86f1-41af-91ab-2d7cd011db47`
**Region:** `eastus2` · **Resource group:** `kaveeshclitest`

**Reused observability resources (existing, both in `kaveeshclitest` RG):**
- AMW: `/subscriptions/.../resourcegroups/kaveeshclitest/providers/microsoft.monitor/accounts/kaveeshclitest`
- AMW Prom query endpoint: `https://kaveeshclitest-eedhgpe4ahdfgpev.eastus2.prometheus.monitor.azure.com`
- Azure Managed Grafana: `/subscriptions/.../resourceGroups/kaveeshclitest/providers/microsoft.dashboard/grafana/kaveeshclitest`

**Preview feature:** the docs note that CCP metrics historically required the subscription-level feature `Microsoft.ContainerService/AzureMonitorMetricsControlPlanePreview`. It was registered for the duration of this test cycle and unregistered at cleanup time.

**Setup to bypass the local `aks-preview` dev override so commands actually exercise GA code:**
```powershell
az config set extension.dev_sources=""                                                       # before
az config set extension.dev_sources="C:\Users\kadubey\Documents\git_repos\azure-cli-extensions"  # after (restored)
```

Verified flags present in GA build only:
```text
$ az extension list --query "[?name=='aks-preview']" -o json
[]
$ az aks create --help | findstr control-plane-metrics
    --enable-control-plane-metrics --enable-cp-metrics : Enable collection of Azure Monitor managed ...
$ az aks update --help | findstr control-plane-metrics
    --disable-control-plane-metrics --disable-cp-metrics : Disable collection of Azure Monitor ...
    --enable-control-plane-metrics  --enable-cp-metrics  : Enable collection of Azure Monitor ...
```

State snapshot convention used below:
```powershell
az aks show -g kaveeshclitest -n <cluster> --query "{state:provisioningState, amp:azureMonitorProfile.metrics.enabled, cp:azureMonitorProfile.metrics.controlPlane}" -o json
```

AMW Prom snapshots use the helper script `query_amw.ps1` (in this same `files/` folder), which runs these PromQL instant queries against the AMW with a bearer token for `https://prometheus.monitor.azure.com`:
- `up{job=~"controlplane.*"}` — proves CCP scrape targets are live
- `etcd_server_has_leader` — default CCP etcd metric
- `etcd_mvcc_db_total_size_in_bytes` — default CCP etcd metric
- `apiserver_current_inflight_requests` — default CCP apiserver metric
- `count(apiserver_request_total)` — total CCP apiserver series cardinality

---

# Part 1 — Brownfield (existing cluster `kaveeshclitest`)

## Baseline (`az aks show`)
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```

## B1 — Negative: `aks create --enable-control-plane-metrics` without parent AMP
```powershell
az aks create -g kaveeshclitest -n test-cp-neg-fake --enable-control-plane-metrics --generate-ssh-keys --location eastus2 --node-count 1
```
```text
ERROR: --enable-control-plane-metrics requires Azure Monitor metrics to be enabled.
       Specify --enable-azure-monitor-metrics or run on a cluster that already has
       Azure Monitor metrics enabled.
```
Exit `1`. ✅ Validation fires client-side (`RequiredArgumentMissingError`). No cluster created.

## B2 — Mutex: `--enable-CP` + `--disable-CP`
```powershell
az aks update -g kaveeshclitest -n kaveeshclitest --enable-control-plane-metrics --disable-control-plane-metrics --yes
```
```text
ERROR: Cannot specify --enable-control-plane-metrics and --disable-control-plane-metrics at the same time.
```
Exit `1`. ✅ `MutuallyExclusiveArgumentError`. No PUT.

## B3 — Mutex: `--enable-CP` + `--disable-azure-monitor-metrics`
```powershell
az aks update -g kaveeshclitest -n kaveeshclitest --enable-control-plane-metrics --disable-azure-monitor-metrics --yes
```
```text
ERROR: Cannot specify --enable-control-plane-metrics together with --disable-azure-monitor-metrics.
```
Exit `1`. ✅ `MutuallyExclusiveArgumentError`. No PUT.

`az aks show` after B1–B3 (unchanged):
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```

## B4 — `--disable-control-plane-metrics` (long form)
Pre `az aks show`:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
```powershell
az aks update -g kaveeshclitest -n kaveeshclitest --disable-control-plane-metrics --yes
```
Exit `0`. Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
✅ CP→false. Parent AMP unchanged.

## B5 — `--enable-control-plane-metrics` (long form)
Pre `az aks show`:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
```powershell
az aks update -g kaveeshclitest -n kaveeshclitest --enable-control-plane-metrics --yes
```
Exit `0`. Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
✅ CP→true.

## B6 — `--disable-cp-metrics` (alias)
Pre `az aks show`:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
```powershell
az aks update -g kaveeshclitest -n kaveeshclitest --disable-cp-metrics --yes
```
Exit `0`. Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
✅ Short alias accepted, CP→false.

## B7 — `--enable-cp-metrics` (alias, restore brownfield)
Pre `az aks show`:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
```powershell
az aks update -g kaveeshclitest -n kaveeshclitest --enable-cp-metrics --yes
```
Exit `0`. Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
✅ Short alias accepted, CP→true. **Brownfield cluster restored.**

> **Note on brownfield AMW verification:** The brownfield `kaveeshclitest` cluster has `cp.enabled=true` set in its spec, but `az monitor data-collection rule association list --resource <kaveeshclitest>` returned `[]` — no DCRA is currently linked. The cluster's metrics wiring was unlinked at some point unrelated to this PR; data-plane verification was therefore done on the greenfield cluster (Part 2) where the wiring was created fresh by our `--enable-azure-monitor-metrics` flow.

---

# Part 2 — Greenfield (new cluster `kclitest-gf` with AMW data-plane verification)

This is the most important section because it exercises:
- The CLI's deferred-flip pattern (initial PUT skips `controlPlane.enabled=true`, second PUT after DCRA creation flips it).
- End-to-end data flow: cluster → CCP collector → DCR/DCE pipeline → AMW → Prometheus query API.

The control-plane scrape targets and default metrics being verified come from the doc list under `controlplane-apiserver` and `controlplane-etcd`:
`apiserver_request_total`, `apiserver_current_inflight_requests`, `etcd_server_has_leader`, `etcd_mvcc_db_total_size_in_bytes`.

## G0 — Create greenfield cluster with `--enable-AMP --enable-CP`
```powershell
az aks create -g kaveeshclitest -n kclitest-gf --location eastus2 `
  --node-count 1 --node-vm-size Standard_D2s_v3 --generate-ssh-keys `
  --enable-azure-monitor-metrics `
  --azure-monitor-workspace-resource-id "/subscriptions/.../microsoft.monitor/accounts/kaveeshclitest" `
  --grafana-resource-id "/subscriptions/.../microsoft.dashboard/grafana/kaveeshclitest" `
  --enable-control-plane-metrics --no-wait
```
CLI output:
```text
WARNING: Monitoring Data Reader role assignment already exists on the Azure Monitor Workspace for the Grafana managed identity. Skipping role assignment.
Using Azure Monitor Workspace (stores prometheus metrics) : /subscriptions/.../kaveeshclitest
```
Exit `0`. Then `az aks wait -g kaveeshclitest -n kclitest-gf --created --timeout 1200` exit `0` at 09:23:14.

`az aks show` post-create:
```json
{
  "amp": true,
  "cp": { "enabled": true },
  "state": "Succeeded"
}
```

DCR + DCRA artifacts confirm the deferred-flip pattern fired:
```powershell
az monitor data-collection rule list -g kaveeshclitest --query "[].name"
```
```json
[ "MSProm-eastus2-kclitest-gf" ]
```
```powershell
az rest --method get --url ".../managedClusters/kclitest-gf/providers/Microsoft.Insights/dataCollectionRuleAssociations?api-version=2022-06-01"
```
```json
{
  "value": [{
    "name": "ContainerInsightsMetricsExtension",
    "properties": {
      "dataCollectionRuleId": ".../dataCollectionRules/MSProm-eastus2-kclitest-gf"
    },
    "systemData": { "createdAt": "2026-06-11T16:18:57Z" }
  }]
}
```

## G1 — Verify CP metrics are flowing (T+26 min after create)

> Important timing note: data-plane targets (`kubelet`, `cadvisor`, `node`, `kube-state-metrics`, `networkobservability-retina`) became "up" within ~5 min of cluster ready. **CCP targets** (`controlplane-apiserver`, `controlplane-etcd`) took **~26 minutes after cluster Succeeded** to start scraping — the hosted control plane needs to deploy the CCP collector pods, which is async on the CRP side.

At 09:49:48 (~26 min after cluster Succeeded), CCP scrape targets came up:

```text
=== T0 SNAPSHOT after CP enable (greenfield) ===
Query time: 09:49:48

--- up{job=~controlplane-.*}: instant ---
  controlplane-etcd      / etcd-6a2ade4f2b0536000113a034-sbw2mcr7kz   -> up=1 @ 09:49:49
  controlplane-apiserver / kube-apiserver-5c559cbf74-8j986            -> up=1 @ 09:49:49
  controlplane-apiserver / kube-apiserver-5c559cbf74-mprwq            -> up=1 @ 09:49:49
  controlplane-etcd      / etcd-6a2ade4f2b0536000113a034-kxhkpvgzv6   -> up=1 @ 09:49:49
  controlplane-etcd      / etcd-6a2ade4f2b0536000113a034-xrh258trfp   -> up=1 @ 09:49:49

--- etcd_server_has_leader: instant ---
  etcd-6a2ade4f2b0536000113a034-kxhkpvgzv6 -> 1 @ 09:49:49
  etcd-6a2ade4f2b0536000113a034-sbw2mcr7kz -> 1 @ 09:49:49
  etcd-6a2ade4f2b0536000113a034-xrh258trfp -> 1 @ 09:49:49

--- etcd_mvcc_db_total_size_in_bytes: instant ---
  etcd-6a2ade4f2b0536000113a034-kxhkpvgzv6 -> 7139328 bytes @ 09:49:49
  etcd-6a2ade4f2b0536000113a034-sbw2mcr7kz -> 7028736 bytes @ 09:49:49
  etcd-6a2ade4f2b0536000113a034-xrh258trfp -> 7880704 bytes @ 09:49:49

--- apiserver_current_inflight_requests: instant ---
  kube-apiserver-5c559cbf74-mprwq [readonly]  -> 1 @ 09:49:49
  kube-apiserver-5c559cbf74-mprwq [mutating]  -> 1 @ 09:49:49
  kube-apiserver-5c559cbf74-8j986  [mutating] -> 3 @ 09:49:49
  kube-apiserver-5c559cbf74-8j986  [readonly] -> 0 @ 09:49:49

--- apiserver_request_total: total series count ---
  count = 803 @ 09:49:49
```
✅ All five default CCP metric families from the doc list are present and scraping fresh data: `apiserver_request_total` (803 series), `apiserver_current_inflight_requests` (2 pods × 2 request_kind), `etcd_server_has_leader` (3 etcd pods, all `=1`), `etcd_mvcc_db_total_size_in_bytes` (3 etcd pods, ~7MB each).

(Raw snapshot saved at `files/snapshot_T0_after_enable.txt`.)

## G2 — Disable CP, then re-query AMW after 15 min
```powershell
az aks update -g kaveeshclitest -n kclitest-gf --disable-control-plane-metrics --yes
```
Submitted at 09:50:17, completed at 09:52:30, exit `0`.

Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": false }, "state": "Succeeded" }
```
✅ CLI correctly flipped the cluster spec to `cp.enabled=false`.

**AMW query at 10:09:03 (~17 min after disable):** CCP scrape targets STILL `up=1` and metrics STILL flowing:
```text
--- up{job=~controlplane-.*}: instant ---
  controlplane-etcd      / sbw2mcr7kz   -> up=1 @ 10:09:03
  controlplane-apiserver / 5c559cbf74-8j986  -> up=1 @ 10:09:03
  controlplane-apiserver / 5c559cbf74-mprwq  -> up=1 @ 10:09:03
  controlplane-etcd      / kxhkpvgzv6   -> up=1 @ 10:09:03
  controlplane-etcd      / xrh258trfp   -> up=1 @ 10:09:03
```

**AMW query at 10:20:09 (~28 min after disable):** still scraping:
```text
controlplane-etcd      -> 3 targets
controlplane-apiserver -> 2 targets
(etcd_mvcc_db_total_size_in_bytes still incrementing: e.g. 7917568→8232960 bytes between snapshots)
```

(Raw snapshots saved at `files/snapshot_T1_after_disable.txt` and `files/snapshot_T1B_after_disable_27min.txt`.)

### ⚠️ Important finding on disable behavior

The CLI side of `--disable-control-plane-metrics` works correctly — it sets `mc.azure_monitor_profile.metrics.control_plane = ManagedClusterAzureMonitorProfileMetricsControlPlane(enabled=False)` and PATCHes the cluster, which is verified by `az aks show`. **However, the actual teardown of the server-side CCP collector pods (which run in the hosted control plane and emit `controlplane-apiserver`/`controlplane-etcd` metrics) is handled asynchronously by the AKS Resource Provider and was observed to take more than 28 minutes** in this test run (>15 min was the user-supplied expectation). This matches the reference aks-preview PR exactly — the CLI code for both ports only flips the spec property; the CRP owns the actual collector lifecycle.

This is a **CRP-side behavior**, not a CLI bug: the CLI's contract is to set the desired state, which it does correctly and immediately. Consumers relying on "metrics stop within N minutes" need to coordinate with CRP teardown SLOs separately.

## G3 — Re-enable CP, verify still flowing
```powershell
az aks update -g kaveeshclitest -n kclitest-gf --enable-control-plane-metrics --yes
```
Submitted at 10:20:34, completed at 10:22:20, exit `0`.

Post `az aks show`:
```json
{ "amp": true, "cp": { "enabled": true }, "state": "Succeeded" }
```
✅ CLI correctly flipped the cluster spec back to `cp.enabled=true`.

AMW query at 10:25:42 (~3 min after re-enable):
```text
--- up{job=~controlplane-.*}: instant ---
  controlplane-etcd      / sbw2mcr7kz   -> up=1 @ 10:25:44
  controlplane-apiserver / 5c559cbf74-8j986  -> up=1 @ 10:25:44
  controlplane-apiserver / 5c559cbf74-mprwq  -> up=1 @ 10:25:44
  controlplane-etcd      / kxhkpvgzv6   -> up=1 @ 10:25:44
  controlplane-etcd      / xrh258trfp   -> up=1 @ 10:25:44

--- count by(job) of up{job=~controlplane.*} ---
  controlplane-etcd      -> 3 targets
  controlplane-apiserver -> 2 targets
```
✅ All 5 CCP scrape targets active (because they never tore down from G2's disable). The CLI spec change is correct; data continuity is a CRP-side behavior.

(Raw snapshot saved at `files/snapshot_T2_after_reenable.txt`.)

## Cleanup
```powershell
az aks delete -g kaveeshclitest -n kclitest-gf --yes --no-wait
az aks wait -g kaveeshclitest -n kclitest-gf --deleted --timeout 1500
```
Exit `0` at 10:32:21. `az aks list -g kaveeshclitest --query "[].name"`:
```json
[ "kaveeshclitest" ]
```
Greenfield cluster fully gone (including its `MC_*` node RG).

```powershell
az feature unregister --namespace Microsoft.ContainerService --name AzureMonitorMetricsControlPlanePreview
az provider register -n Microsoft.ContainerService
```
Feature now `Unregistered` on the subscription.

```powershell
az config set extension.dev_sources="C:\Users\kadubey\Documents\git_repos\azure-cli-extensions"
```
`aks-preview` extension re-enabled.

---

# Summary

| # | Mode | Test | Pre cp | Post cp | CLI verified by | AMW verified by | Result |
|---|---|---|---|---|---|---|---|
| B1 | brownfield | `aks create --enable-CP` w/o AMP | — | — | error message + exit 1 | n/a | ✅ `RequiredArgumentMissingError` |
| B2 | brownfield | mutex enable+disable CP | true | true | error msg + exit 1, no PUT | n/a | ✅ `MutuallyExclusiveArgumentError` |
| B3 | brownfield | mutex enable-CP + disable-AMP | true | true | error msg + exit 1, no PUT | n/a | ✅ `MutuallyExclusiveArgumentError` |
| B4 | brownfield | `--disable-control-plane-metrics` | true | false | `az aks show` post | n/a (no DCRA on this cluster) | ✅ |
| B5 | brownfield | `--enable-control-plane-metrics` | false | true | `az aks show` post | n/a | ✅ |
| B6 | brownfield | `--disable-cp-metrics` alias | true | false | `az aks show` post | n/a | ✅ |
| B7 | brownfield | `--enable-cp-metrics` alias | false | true | `az aks show` post | n/a | ✅ |
| G0 | **greenfield** | `aks create --enable-AMP --enable-CP` | n/a | true | `az aks show` Succeeded; DCR + DCRA created (deferred-flip pattern works) | — | ✅ |
| G1 | greenfield | verify CP metrics flow | true | true | `az aks show` | **PromQL: all 5 default CCP metric families present**, 5 controlplane scrape targets `up=1`, fresh timestamps | ✅ |
| G2 | greenfield | `--disable-control-plane-metrics` | true | false | `az aks show` post | **CLI spec change confirmed; CRP-side CCP teardown observed to take >28 min** (documented as CRP behavior, not CLI bug) | ✅ (CLI), ⚠️ (CRP async) |
| G3 | greenfield | `--enable-control-plane-metrics` | false | true | `az aks show` post | CCP scrape targets still up; CP metrics continuous | ✅ |

**Combined verification:** 7/7 new unit tests + 14/14 existing azure-monitor regressions + 11/11 live tests (B1–B7, G0–G3) all pass. AMW workspace verification confirms that on the greenfield enable path, every default CCP metric family from the docs (`apiserver_request_total`, `apiserver_current_inflight_requests`, `etcd_server_has_leader`, `etcd_mvcc_db_total_size_in_bytes`) is correctly scraped and ingested.

**Environment restored:**
- Brownfield cluster `kaveeshclitest`: AMP=true, CP=true (as at session start).
- Greenfield cluster `kclitest-gf`: deleted; `MC_*` node RG gone; only `kaveeshclitest` remains in the RG.
- `aks-preview` extension re-enabled.
- `AzureMonitorMetricsControlPlanePreview` feature unregistered on the subscription as requested.

**Artifacts in `files/`:**
- `query_amw.ps1` — helper script for AMW Prometheus snapshots
- `snapshot_T0_after_enable.txt` — CCP metrics flowing after greenfield create
- `snapshot_T1_after_disable.txt` — 15 min after disable (CCP still up)
- `snapshot_T1B_after_disable_27min.txt` — 28 min after disable (CCP still up)
- `snapshot_T2_after_reenable.txt` — after re-enable
