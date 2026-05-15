# Problem Statement

Customer statement: "The vote app, running on port 31000, stopped working."

Scope: Kubernetes `default/vote` application exposed as a NodePort service on `31000`.

Impact: the vote frontend was unavailable or unstable during the observed failure window. The strongest evidence points to the only active `vote` pod being evicted from node `xp10-vm22` when that node entered `DiskPressure`; Kubernetes then created replacement pods until one became healthy on `xp10-vm21`.

Time window: Kubernetes evidence is in UTC. The key failure window is `2024-08-30 05:56:47Z` through at least `2024-08-30 06:01:51Z`.

# Root Cause Analysis

Most defensible conclusion: the vote app outage was caused by ephemeral-storage exhaustion on Kubernetes node `xp10-vm22`, which triggered kubelet `DiskPressure` and evicted the running `vote` pod.

Facts:

- The `vote` service existed as a NodePort on `31000`: `default vote NodePort 10.106.16.13 ... 5000:31000/TCP`.
- The original `vote` pod `vote-5599b5ffb5-5jbkc` was scheduled to `xp10-vm22`.
- At `2024-08-30T05:56:47Z`, kubelet evicted that pod because ephemeral storage was nearly exhausted: threshold `7883388831`, available `82836Ki`.
- Node `xp10-vm22` reported `DiskPressure=True` with `KubeletHasDiskPressure`.
- Host disk evidence confirms `/dev/sda2` on `xp10-vm22` was `99%` full with only `576M` available.
- `xp10-vm20` and `xp10-vm21` were only `23%` used on `/dev/sda2`, so the pressure was node-specific.
- After the eviction, Kubernetes successfully scheduled `vote-5599b5ffb5-4xr6s` on `xp10-vm21`; logs show Gunicorn listening on port `80` and repeated HTTP `200` responses.

Inference:

- The customer-facing failure on port `31000` is best explained by the `vote` pod being killed during node disk pressure and by repeated replacement pod churn on the unhealthy node.
- By the time the bundle was collected, the deployment had partially self-healed: `vote-5599b5ffb5-4xr6s` was Running and the `vote` endpoint pointed to `10.244.1.189:80`. However, `xp10-vm22` still had active disk pressure and continued eviction-threshold events, so recurrence risk remained high.

Confidence: High that node ephemeral-storage exhaustion caused the observed `vote` pod loss and service instability. Medium-high for the complete customer-visible outage path because the bundle does not include an external curl/browser failure, packet capture, or load-balancer health check from the customer's client path.

Underlying disk consumer: Unknown. The bundle includes `df` output proving the node filesystem is full, but not enough directory-level evidence such as `du`, containerd image usage, kubelet eviction logs, or log volume breakdown to prove which file tree filled the disk.

## Alternative Theories Considered

| ID | Hypothesis | Status | Evidence and reasoning |
| --- | --- | --- | --- |
| H1 | `xp10-vm22` disk pressure evicted the `vote` pod and caused the outage | Confirmed as the best-supported cause | Direct eviction event, node `DiskPressure=True`, and host filesystem at `99%` full all align in time. |
| H2 | The `vote` Service or NodePort was misconfigured | Rejected | Service exists as NodePort `31000`, selector is `app=vote`, `externalTrafficPolicy=Cluster`, and endpoint exists after recovery. |
| H3 | Kubernetes networking or kube-proxy was down | Weak / not supported | Nodes were `Ready`; kube-proxy pods were running and ready. Router log search found no clear link-down/drop/reset pattern explaining the app failure. |
| H4 | Application backend failure, Redis, DB, or worker caused the vote page outage | Possible secondary issue, not the main port-31000 outage | Redis and DB were running; worker initially had image pull trouble and result logs showed `relation "votes" does not exist`, but the `vote` frontend later served HTTP `200`. These findings may affect vote processing/results, not the primary loss of the `vote` web endpoint. |

# Key Evidence

## Kubernetes service and endpoint state

- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/kubectl_get_services.txt:6`
  - `default vote NodePort 10.106.16.13 <none> 5000:31000/TCP`
- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/kubectl_get_endpoints.txt:6`
  - `default vote 10.244.1.189:80`
- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/cluster-state/default/services.json`
  - `vote` selector is `app=vote`, NodePort is `31000`, and `externalTrafficPolicy` is `Cluster`.

Why this matters: the service object itself was present and correctly mapped external port `31000` to the vote pod target port. This weakens a service-misconfiguration theory.

## Vote pod churn and eviction

- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/kubectl_get_pods.txt`
  - Many `vote-5599b5ffb5-*` pods were `Evicted` around `5m` age.
  - `vote-5599b5ffb5-4xr6s` was the only running vote pod at collection time.
- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/cluster-state/default/events.json:1044`
  - `The node was low on resource: ephemeral-storage. Threshold quantity: 7883388831, available: 82836Ki. Container vote was using 268Ki...`
- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/cluster-state/default/events.json:4255`
  - `Node xp10-vm22 status is now: NodeHasDiskPressure`
- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/cluster-state/default/events.json:4281`
  - `FreeDiskSpaceFailed`; kubelet attempted to free about `9907376947` bytes but found no eligible images to garbage collect.

Why this matters: this is direct Kubernetes evidence that the node had insufficient local ephemeral storage and that kubelet took eviction actions against the `vote` workload.

## Node filesystem state

- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/servers/xp10-vm22/df.log:3`
  - `/dev/sda2 49G 47G 576M 99% /`
- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/servers/xp10-vm20/df.log:3`
  - `/dev/sda2 49G 11G 37G 23% /`
- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/servers/xp10-vm21/df.log:3`
  - `/dev/sda2 49G 11G 37G 23% /`
- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/cluster-state/nodes.json`
  - `xp10-vm22` has taint `node.kubernetes.io/disk-pressure:NoSchedule` and `DiskPressure=True`.

Why this matters: the disk pressure was real at the host layer and isolated to `xp10-vm22`, matching the pod eviction location.

## Recovery signal

- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/cluster-state/default/events.json:680`
  - `Successfully assigned default/vote-5599b5ffb5-4xr6s to xp10-vm21`
- `work/EXP11_20240830_073516/EXP11_20240830_073516/syslogs/kubernetes/default/vote-5599b5ffb5-4xr6s/logs.txt:2-3`
  - Gunicorn started and listened at `http://0.0.0.0:80`.
- `work/EXP11_20240830_073516/EXP11_20240830_073516/syslogs/kubernetes/default/vote-5599b5ffb5-4xr6s/logs.txt:9`
  - HTTP `GET /` returned `200` at `2024-08-30 05:56:52 +0000`.

Why this matters: the app recovered once the replacement pod ran on a healthy node, which further supports node-local disk pressure rather than a broken image, service, or app binary.

## Secondary findings

- `work/EXP11_20240830_073516/EXP11_20240830_073516/showcommands/kubernetes/cluster-state/default/events.json`
  - `worker-7dd74bcbbb-cpvtw` initially hit `ErrImagePull` / `ImagePullBackOff` on `xp10-vm22` after a connection reset while pulling `dockersamples/examplevotingapp_worker`.
- `work/EXP11_20240830_073516/EXP11_20240830_073516/syslogs/kubernetes/default/result-7845d79c55-vln9t/logs.txt`
  - The result app logged repeated `relation "votes" does not exist`.

Why this matters: these are likely secondary or adjacent application issues. They may affect vote processing or result display, but they do not explain the direct `vote` frontend availability failure on NodePort `31000` as well as the disk-pressure eviction evidence does.

# Timeline

All times are UTC.

| Time | Event |
| --- | --- |
| 2024-08-30T05:38:38Z | App pods and services are created in the `default` namespace. |
| 2024-08-30T05:38:44Z | Original `vote-5599b5ffb5-5jbkc` is pulling on `xp10-vm22`. |
| 2024-08-30T05:41:25Z | `worker-7dd74bcbbb-cpvtw` image pull fails on `xp10-vm22` due a connection reset while downloading from Docker registry. |
| 2024-08-30T05:45:20Z | Original vote pod starts on `xp10-vm22`. |
| 2024-08-30T05:56:47Z | Kubelet evicts the original vote pod because node ephemeral storage is below threshold. |
| 2024-08-30T05:56:48Z to 2024-08-30T05:56:52Z | Many replacement `vote` and `worker` pods scheduled to `xp10-vm22` are immediately evicted due `DiskPressure`. |
| 2024-08-30T05:56:50Z | Replacement `vote-5599b5ffb5-4xr6s` starts on `xp10-vm21`. |
| 2024-08-30T05:56:52Z | `xp10-vm22` reports `NodeHasDiskPressure`; the new vote pod on `xp10-vm21` begins returning HTTP `200`. |
| 2024-08-30T05:57:00Z | Kubelet reports `FreeDiskSpaceFailed` on `xp10-vm22`: attempted to free about `9.9 GB`, but found `0` bytes eligible. |
| 2024-08-30T06:01:51Z | `xp10-vm22` continues reporting `EvictionThresholdMet`, attempting to reclaim ephemeral storage. |

# Suggested Steps for Resolving the Issue

## Immediate Mitigation

1. Keep production traffic away from `xp10-vm22` until disk pressure is cleared.
   - `cordon xp10-vm22` to prevent new pods from landing there.
   - If safe for the environment, drain non-critical workloads from `xp10-vm22`.
2. Free space on `xp10-vm22`.
   - Identify large directories under `/var/lib/containerd`, `/var/log`, `/var/lib/kubelet`, and application log paths.
   - Rotate or truncate runaway logs only after preserving any needed evidence.
   - Prune unused container images through supported runtime tooling if appropriate.
3. Ensure the vote deployment has at least one healthy replica on a non-pressured node.
   - If service impact continues, temporarily scale `vote` above one replica after confirming healthy node capacity.
4. Test NodePort access explicitly from the customer path:
   - `http://172.16.0.82:31000/`
   - `http://172.16.0.86:31000/`
   - `http://172.16.0.90:31000/`

## Permanent Resolution

1. Find and remove the underlying disk consumer on `xp10-vm22`.
   - Collect `du -xhd1 /`, `du -xhd1 /var`, `du -xhd1 /var/lib`, containerd image/container usage, and kubelet logs around `2024-08-30T05:56Z`.
   - Confirm whether the growth is from images, writable container layers, logs, support bundles, or another host process.
2. Add guardrails for node ephemeral storage.
   - Set pod `requests` and `limits` for `ephemeral-storage` where practical.
   - Configure log rotation and container runtime garbage collection thresholds.
   - Monitor node filesystem usage and kubelet eviction signals with alerting before disk reaches eviction thresholds.
3. Improve application availability.
   - Run more than one `vote` replica and spread replicas across nodes with topology spread constraints or anti-affinity.
   - Avoid single-replica exposure for user-facing NodePort services.
4. Investigate secondary app health.
   - Confirm the `worker` creates the `votes` table and that the result app stops logging `relation "votes" does not exist`.
   - Validate end-to-end vote submission, Redis queueing, worker processing, and result display.

## Validation Steps

1. Verify `xp10-vm22` clears pressure:
   - `kubectl describe node xp10-vm22` shows `DiskPressure=False`.
   - The `node.kubernetes.io/disk-pressure:NoSchedule` taint is removed.
   - `df -h /` shows adequate free space.
2. Verify vote service health:
   - `kubectl get deploy vote` shows desired replicas available.
   - `kubectl get endpoints vote` shows healthy endpoints.
   - Curl port `31000` through each node IP and through the customer access path.
3. Verify no recurrence:
   - No new `Evicted`, `EvictionThresholdMet`, or `FreeDiskSpaceFailed` events.
   - Node disk usage remains below alert thresholds for a sustained observation window.

## Remaining Risks or Unknowns

- The exact disk consumer on `xp10-vm22` is not identified in the provided bundle.
- The bundle does not include customer-side HTTP errors, packet captures, ingress/load-balancer logs, or direct curl output for port `31000`, so the external access path cannot be fully proven from this data alone.
- The result/worker/DB evidence suggests a possible secondary application readiness issue that should be validated after restoring node health.
