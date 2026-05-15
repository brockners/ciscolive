# Problem Statement

Customer statement: "The vote app, running on port 31000, stopped working."

Scope and impact: the Kubernetes `vote` Service exists as a NodePort on port `31000`, but it has no ready backend endpoints. Traffic to the app cannot be served while the Service endpoint list is empty.

Time window: the strongest evidence places the failure around `2024-05-02T11:39Z` to `2024-05-02T11:46Z`. Kubernetes objects use UTC timestamps. Host journal entries are local time, shown as `Mai 02 13:41`, which corresponds to approximately `2024-05-02T11:41Z`.

# Root Cause Analysis

Most likely root cause, high confidence: the vote application became unavailable because the only vote replica was on Kubernetes node `xp06-vm22`, and that node became unreachable from the cluster after its upstream router path was administratively shut down on R4.

The failure chain is:

1. The `vote` Service was still present and exposed as `5000:31000/TCP`, so the NodePort object itself was not missing.
2. The Service had no endpoints, meaning Kubernetes had no ready pod selected by the Service.
3. All observed `vote` pods were scheduled on `xp06-vm22`; the original running pod entered `Terminating` after the node went `NotReady`, while replacement pods were also not available.
4. `xp06-vm22` stopped posting node status and was tainted `node.kubernetes.io/unreachable`.
5. The kubelet on `xp06-vm22` was still running, but its logs show repeated failures reaching the API server at `172.16.0.82:6443`, changing from timeouts to `connect: no route to host`.
6. R4, the router adjacent to `xp06-vm22`, had its node-facing interface up but both routed uplinks shut or down. Its route table contained only connected routes and no route toward the API server. R4 logs/debugs show traffic from `172.16.0.86` to `172.16.0.82` being treated as unroutable.

Alternative theories considered:

| ID | Hypothesis | Status | Reason |
| --- | --- | --- | --- |
| H1 | R4 underlay isolation caused `xp06-vm22` to become `NotReady`, leaving `vote` with no ready endpoints. | Confirmed / high confidence | Explains the NodeNotReady event, kubelet API failures, empty Service endpoints, and R4 unroutable packets. |
| H2 | The `vote` Service or NodePort was misconfigured. | Rejected | The Service exists with `nodePort: 31000`, port `5000`, target port `80`, and selector `app: vote`. The failure is endpoint availability, not service definition. |
| H3 | The vote container crashed or the application failed internally. | Weak / rejected with current evidence | The original vote container was shown as started and container-ready before pod readiness was lost due to node state. No CrashLoopBackOff or OOM evidence was found for vote. |
| H4 | Node resource exhaustion caused the outage. | Weak / rejected with current evidence | Disk and memory snapshots do not show exhaustion, and the kubelet process was still running. The dominant failure is network reachability to the API server. |
| H5 | Scheduler behavior kept replacement pods on an unreachable node. | Possible secondary symptom | Pending and terminating replacement pods remained assigned to `xp06-vm22`, but this does not explain why the node became unreachable. |

# Key Evidence

## Kubernetes Service and Endpoint State

File: `data/EXP09_20240502_133631/k8s_logs/kubectl_get_services.txt`

Relevant summary: `default vote NodePort 10.109.127.236 <none> 5000:31000/TCP 14m`.

Why it matters: the NodePort service still exists on port `31000`; the customer-facing object was not deleted.

File: `data/EXP09_20240502_133631/k8s_logs/kubectl_get_endpoints.txt`

Relevant summary: `default vote <none> 14m`.

Why it matters: the direct Kubernetes reason for the app being unavailable is that the Service has no ready endpoints.

File: `data/EXP09_20240502_133631/k8s_logs/kubectl_cluster-info_dump.txt`

Relevant summary: the `vote` Service has selector `app: vote`, `nodePort: 31000`, port `5000`, and `targetPort: 80`.

Why it matters: this refutes a simple NodePort or selector mismatch as the primary fault.

## Vote Pods and Node State

File: `data/EXP09_20240502_133631/k8s_logs/kubectl_get_pods.txt`

Relevant summary:

```text
vote-79b55466bc-4v9mk  0/1  Pending      95s
vote-79b55466bc-g4s9v  0/1  Terminating  6m35s
vote-79b55466bc-l6r9b  1/1  Terminating  14m
```

Why it matters: no `vote` pod was in a normal available state.

File: `data/EXP09_20240502_133631/k8s_logs/kubectl_get_nodes.txt`

Relevant summary: `xp06-vm20` and `xp06-vm21` were `Ready`; `xp06-vm22` was `NotReady`.

Why it matters: the issue was isolated to the node that hosted the vote workload.

File: `data/EXP09_20240502_133631/k8s_logs/kubectl_describe_pods.txt`

Relevant summary:

- `vote-79b55466bc-4v9mk` was assigned to `xp06-vm22` and was `Pending`.
- `vote-79b55466bc-g4s9v` was assigned to `xp06-vm22` and was `Terminating`.
- `vote-79b55466bc-l6r9b` was assigned to `xp06-vm22/172.16.0.86`, had pod IP `10.244.1.82`, and showed `Warning NodeNotReady ... Node is not ready`.

Why it matters: Kubernetes had no healthy vote backend because the vote pods were tied to the NotReady node.

File: `data/EXP09_20240502_133631/k8s_logs/kubectl_cluster-info_dump.txt`

Relevant summary:

- `xp06-vm22` condition changed to `Ready: Unknown` with reason `NodeStatusUnknown` and message `Kubelet stopped posting node status`.
- `xp06-vm22` was tainted with `node.kubernetes.io/unreachable`.
- The `vote` Deployment became unavailable with reason `MinimumReplicasUnavailable`.
- Taint manager marked `vote-79b55466bc-l6r9b` for deletion at `2024-05-02T11:46:18Z`.

Why it matters: the endpoint loss tracks directly to node reachability and Kubernetes eviction behavior.

## Node-Level Evidence

File: `data/EXP09_20240502_133631/server_logs/xp06-vm22/journactl.log`

Relevant summary:

```text
kubelet failed to update lease ... https://172.16.0.82:6443 ... Client.Timeout exceeded while awaiting headers
kubelet failed to patch status ... Client.Timeout exceeded while awaiting headers
http2: client connection lost
dial tcp 172.16.0.82:6443: connect: no route to host
```

Why it matters: `xp06-vm22` lost network reachability to the API server. The transition from timeout to `no route to host` is a strong network-layer signal.

File: `data/EXP09_20240502_133631/server_logs/xp06-vm22/ps_aux.txt`

Relevant summary: the `kubelet` process was still present.

Why it matters: the node was not `NotReady` simply because kubelet had exited.

File: `data/EXP09_20240502_133631/server_logs/xp06-vm22/df.txt`

Relevant summary: root filesystem usage was about `23%`.

Why it matters: no disk-full condition was evident.

File: `data/EXP09_20240502_133631/server_logs/xp06-vm22/meminfo.txt`

Relevant summary: `MemAvailable` was about `1075136 kB`.

Why it matters: available evidence does not support memory exhaustion as the primary cause.

## Router and Underlay Evidence

File: `data/EXP09_20240502_133631/routers_data/R4_show_run.json`

Relevant summary:

- `GigabitEthernet0/2` is described as `R4 to alpine86 via Gi0/2 (EXT-CONN)` with `ip address 172.16.0.85 255.255.255.252`.
- `GigabitEthernet0/0` has `ip address 172.16.0.50 255.255.255.252` and `shutdown`.
- `GigabitEthernet0/1` has `ip address 172.16.0.53 255.255.255.252` and `shutdown`.

Why it matters: R4 still had the node-facing segment up, but its routed uplinks were administratively shut.

File: `data/EXP09_20240502_133631/routers_data/R4_show_ip_interface_brief.json`

Relevant summary:

- `GigabitEthernet0/0` was `administratively down/down`.
- `GigabitEthernet0/1` was `administratively down/down`.
- `GigabitEthernet0/2` was `up/up`.

Why it matters: this confirms the topology state at collection time.

File: `data/EXP09_20240502_133631/routers_data/R4_show_ip_route.json`

Relevant summary: R4 had only connected/local routes for `10.0.0.4/32`, `172.16.0.84/30`, and `172.16.100.0/24`; no OSPF-learned routes and no default route were present.

Why it matters: R4 could not route off the `xp06-vm22` segment toward the API server.

File: `data/EXP09_20240502_133631/routers_data/R4_tech-support.txt`

Relevant summary: event trace records `IOSv[Gi0/1]: Shutdown`, followed by physical/controller shutdown activity. Later debug output shows packets sourced from `172.16.0.86` toward `172.16.0.82` being marked `unroutable`.

Why it matters: this directly links R4 interface shutdown to dropped reachability between `xp06-vm22` and the API server.

File: `data/EXP09_20240502_133631/routers_data/R6_log_20240502-113621`

Relevant summary: OSPF neighbor `10.0.0.4` on `GigabitEthernet0/2` moved from `FULL` to `DOWN` due to dead timer expiry around `May 2 11:39:42`.

Why it matters: R6 observed loss of the R4 adjacency immediately before Kubernetes marked `xp06-vm22` unreachable.

# Timeline

| Time | Source | Event |
| --- | --- | --- |
| `2024-05-02 11:38:57` | `R4_tech-support.txt` | R4 `GigabitEthernet0/1` shutdown event recorded. |
| `2024-05-02 11:39:42` | `R6_log_20240502-113621` | R6 OSPF neighbor to R4 went from `FULL` to `DOWN` due to dead timer expiry. |
| `2024-05-02 11:40:55Z` approx | `xp06-vm22/journactl.log` | Kubelet began failing to update lease/status against API server `172.16.0.82:6443`. |
| `2024-05-02 11:41:13Z` | `kubectl_cluster-info_dump.txt` | Kubernetes marked `xp06-vm22` `Ready: Unknown` / `NodeStatusUnknown`. |
| `2024-05-02 11:41:13Z` | `kubectl_cluster-info_dump.txt` | `vote` Deployment became unavailable with `MinimumReplicasUnavailable`. |
| `2024-05-02 11:46:18Z` | `kubectl_cluster-info_dump.txt` | Taint manager marked original vote pod for deletion due to unreachable node. |
| Collection time | `kubectl_get_endpoints.txt` | `vote` endpoint list was `<none>`. |

# Suggested Steps for Resolving the Issue

## Immediate Mitigation

1. Restore R4 underlay connectivity for the `xp06-vm22` segment. At minimum, verify the intended state of R4 `GigabitEthernet0/1` and re-enable it if it should be active:

   ```text
   configure terminal
   interface GigabitEthernet0/1
   no shutdown
   ```

   Also validate whether `GigabitEthernet0/0` should remain shut. The evidence shows both R4 routed uplinks were shut/down while the node-facing interface stayed up.

2. If service recovery is more urgent than node recovery, move the workload off `xp06-vm22`:

   ```text
   kubectl cordon xp06-vm22
   kubectl delete pod -l app=vote
   kubectl get pods -l app=vote -o wide
   kubectl get endpoints vote
   ```

   If pods remain stuck terminating because the node is unreachable, force deletion may be required, but only after accepting the risk that the isolated node may still be running the old container until connectivity is restored.

3. Temporarily scale `vote` above one replica if capacity allows, and spread replicas across healthy nodes:

   ```text
   kubectl scale deployment vote --replicas=2
   ```

## Permanent Resolution

1. Correct and save the intended R4 interface configuration so the node subnet `172.16.0.84/30` has a routed path toward the Kubernetes API server and the rest of the cluster.
2. Verify OSPF adjacency restoration between R4 and its upstream routers, especially the R4-R6 adjacency.
3. Add resilience for the `vote` Deployment:
   - run at least two replicas,
   - use topology spread constraints or pod anti-affinity across nodes,
   - consider a PodDisruptionBudget if the app is expected to stay available during node failures.
4. Add monitoring alerts for:
   - `NodeNotReady` or `NodeStatusUnknown`,
   - Service endpoint count dropping to zero,
   - OSPF neighbor loss or critical interface shutdown on R4.

## Validation Steps

Network validation:

```text
show ip interface brief
show ip route 172.16.0.82
show ip ospf neighbor
ping 172.16.0.82 source 172.16.0.85
```

Node and Kubernetes validation:

```text
kubectl get nodes
kubectl get pods -l app=vote -o wide
kubectl get endpoints vote
kubectl describe deployment vote
curl http://<node-ip>:31000/
```

Expected successful state:

- `xp06-vm22` returns to `Ready`, or vote pods are running on another `Ready` node.
- `kubectl get endpoints vote` shows at least one pod IP and port.
- NodePort `31000` returns the vote app.
- R4 has a route to `172.16.0.82` and OSPF adjacency is restored.

## Remaining Risks or Unknowns

- The evidence shows R4 interface shutdown and routing loss, but it does not identify who or what issued the configuration change. The R4 logs include configuration activity from `cisco` via `vty0 (172.16.100.100)`, but operator intent is unknown.
- The exact application-level vote logs were not retrievable from the affected node because the API server could not reach kubelet on `172.16.0.86:10250`.
- Replacement vote pod placement should be reviewed after node recovery. If the deployment has only one replica, a single node failure can still remove all vote endpoints.
