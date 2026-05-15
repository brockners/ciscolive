---
name: kubernetes-troubleshooting
description: "Evidence-driven troubleshooting for Kubernetes and OpenShift deployment evidence bundles, tarballs, must-gather data, kubectl outputs, pod/container logs, events, manifests, Helm/Kustomize files, node data, CNI/DNS/ingress/storage artifacts, and support bundles. Use when diagnosing application deployment failures, CrashLoopBackOff, ImagePullBackOff, Pending pods, probe failures, service or ingress timeouts, DNS failures, kubelet/node issues, CNI networking, metrics-server/APIService problems, PVC/CSI issues, or rollout/change correlation."
---

# Kubernetes Troubleshooting

## Mission

Troubleshoot Kubernetes evidence like a senior escalation engineer. Preserve customer artifacts, start broad, maintain competing hypotheses, correlate application symptoms with cluster state, and separate directly observed facts from inferences and unknowns.

## Intake

1. Treat the customer's `data/` directory and uploaded bundles as read-only evidence.
2. List supplied files, sizes, timestamps, archive types, and top-level archive contents before deep analysis.
3. Extract archives into scratch space such as `work/<bundle-name>/`, preserving directory structure. Unpack nested archives into the same scratch tree.
4. Build a quick inventory of pod logs, events, `kubectl get/describe` outputs, manifests, Helm/Kustomize files, node and kubelet data, CNI/DNS/ingress/storage artifacts, metrics, alerts, README/index files, and collection metadata.
5. Identify cluster name, namespaces, workloads, pods, services, nodes, IPs, container images, Kubernetes version, CNI, ingress controller, CSI driver, and collection time. If any identity is inferred from filenames or log content, label it as an inference.
6. Read summaries, support-bundle analysis output, must-gather indexes, README files, and collection manifests before raw logs.

## Useful Tool Choices

Prefer simple read-only tools first: `rg`, `find`/`Get-ChildItem`, `tar -tf`, `tar -xf` into scratch, `unzip -l`, `jq`, `yq`, `awk`, and `sed`.

Use structured parsing when available:

- `jq` for Kubernetes JSON output.
- `yq` or a YAML parser for manifests, Helm output, and support-bundle specs.
- `kubectl` only against offline files unless the user explicitly wants live-cluster checks and access is available.
- Packet-analysis tools for `.pcap` or `.pcapng` only after determining relevant pods, nodes, services, protocols, and time windows.

If a parser fails, keep raw text as the source of truth and note the parser limitation.

## Artifact Classification

For every major file, classify the artifact before interpreting it:

- Collection metadata: README, manifest, index, support-bundle analysis, must-gather metadata, timestamps.
- Resource state: `kubectl get` output, API resources, node/pod/service/deployment/replicaset/statefulset/daemonset/PVC/PV/APIService status.
- Descriptions and events: `kubectl describe`, namespace events, pod scheduling events, probe failures, volume attach/mount events.
- Logs: application containers, init containers, previous container logs, control plane, kubelet, container runtime, CNI, CoreDNS, ingress, CSI, metrics-server.
- Manifests and configuration: Deployment, StatefulSet, DaemonSet, Job, Service, Ingress/Route/Gateway, NetworkPolicy, HPA, PDB, ConfigMap, Secret metadata, Helm values, Kustomize overlays.
- Node/system evidence: node conditions, kubelet logs, container runtime logs, kernel messages, disk/memory/CPU pressure, network interface data.
- Network evidence: Services, Endpoints/EndpointSlices, DNS logs, CNI logs, kube-proxy/IPVS/iptables, NetworkPolicy, ingress/load balancer logs, captures.
- Storage evidence: PVC/PV binding, StorageClass, CSI controller/node logs, mount/attach errors, filesystem and quota messages.

## Broad First Pass

Run broad searches before focusing on one theory. Useful keyword groups:

```text
error|fail|failed|timeout|timed out|refused|reset|unreachable|no route to host|i/o timeout|context deadline exceeded
CrashLoopBackOff|ImagePullBackOff|ErrImagePull|CreateContainerConfigError|RunContainerError|BackOff|OOMKilled|Evicted|Pending
probe failed|readiness|liveness|startup probe|unhealthy|restart|exit code|signal|panic|segfault|exception|stack trace
FailedScheduling|Insufficient|taint|toleration|node affinity|pod affinity|preemption|NotReady|Unknown|DiskPressure|MemoryPressure|PIDPressure
DNS|CoreDNS|SERVFAIL|NXDOMAIN|plugin/errors|lookup|no such host|connection refused|TLS|certificate|x509
Endpoints|EndpointSlice|no endpoints|Service|Ingress|Route|Gateway|load balancer|503|504|upstream|connection reset
CNI|Calico|Cilium|Flannel|OVN|Weave|kube-proxy|iptables|IPVS|NetworkPolicy|MTU|drop|conntrack
PVC|PV|StorageClass|CSI|mount|attach|detach|volume|filesystem|read-only|permission denied|disk full|inode
APIService|metrics.k8s.io|metrics-server|kubelet|10250|apiserver|etcd|leader|watch|reflector
```

Capture context around hits, not isolated lines. Collapse repeated noisy streams by `(component, message pattern, affected object, time range)` so high-volume errors do not hide sparse causal clues.

## Hypothesis Loop

After the first pass, maintain 2-5 competing hypotheses. Use this table shape in notes and final output when useful:

| ID | Hypothesis | Status | Confidence | Supporting evidence | Refuting evidence | Next check |
| --- | --- | --- | --- | --- | --- | --- |

Allowed statuses: `Open`, `Strengthening`, `Weakening`, `Rejected`, `Confirmed`.

For each hypothesis, search for support and refutation. Stop when one hypothesis explains the observed symptoms materially better than alternatives and the evidence is consistent across logs, resource state, events, config, and timeline.

## Kubernetes Lenses

### Application Versus Cluster

Separate primary application failures from platform fallout. An application log such as `Waiting for db` or `relation does not exist` is a symptom until correlated with Service/Endpoint state, DNS, network reachability, database pod logs, migrations, configuration, and rollout history.

Use this dependency chain for application deployment issues:

1. Workload desired state: Deployment/StatefulSet/Job spec, rollout status, replica counts, generation/observedGeneration.
2. Pod lifecycle: phase, container states, restart counts, termination reason, exit code, previous logs.
3. Scheduling and node placement: events, node selectors, affinity, taints, resource requests, topology spread, node readiness.
4. Container start: image pull, credentials, architecture, command/args, env/config, secrets, volumes, permissions.
5. Health checks: readiness, liveness, startup probes, probe target, timing, dependency readiness.
6. Dependencies: Service, Endpoints, DNS, NetworkPolicy, ingress, database/cache/message-bus availability.
7. Recent changes: image tag, config, secret, Helm release, manifest diff, rollout time, node maintenance.

### Pods And Controllers

For `CrashLoopBackOff`, check previous container logs, exit code, termination reason, start time, restart cadence, probes, config/env, mounted files, command/args, image version, and dependency readiness.

For `ImagePullBackOff` or `ErrImagePull`, check exact image reference, tag existence, registry DNS/TLS/auth, imagePullSecrets, rate limits, pull policy, node architecture, and private registry reachability.

For `Pending`, check `FailedScheduling` events, resource requests, taints/tolerations, node selectors/affinity, topology constraints, PVC binding, runtime class, quotas, and PodSecurity admission.

For `CreateContainerConfigError`, check missing ConfigMaps/Secrets, invalid env references, projected volumes, security context, and malformed command/args.

### Services, DNS, And Ingress

Do not diagnose DNS from application errors alone. Correlate:

- CoreDNS logs and readiness.
- `Service` selector and port/targetPort.
- `Endpoints` or `EndpointSlice` presence and readiness.
- Pod labels matching selectors.
- NetworkPolicy, CNI, kube-proxy, and node reachability.
- Ingress/Gateway/Route backend selection and controller logs.

CoreDNS timeouts to an upstream nameserver indicate upstream DNS or node/network reachability issues. NXDOMAIN for cluster service names often points to namespace/name/search-path mistakes or missing services. Empty Endpoints usually points to selector mismatch or unready pods.

### Node, Kubelet, And Metrics

Treat kubelet connectivity errors as node-path evidence, not just metrics noise. Repeated failures to reach `https://<nodeIP>:10250` can explain missing logs, failed `kubectl logs`, metrics-server scrape failures, and API server proxy errors.

Correlate kubelet/node symptoms with:

- Node conditions: `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`, `NetworkUnavailable`.
- Lease updates and `NodeNotReady` events.
- Kubelet and container runtime logs.
- CNI daemon logs on the same node.
- Control-plane errors proxying to kubelet.
- Network reachability between control plane, metrics-server, and node IPs.

### CNI And Pod Networking

For pod-to-pod, pod-to-service, or API reachability failures, check CNI daemon logs, node routes, overlay health, MTU, NetworkPolicy, kube-proxy/IPVS/iptables, conntrack exhaustion, and whether failures concentrate on one node, namespace, or CIDR.

If CNI logs cannot watch Kubernetes resources because `https://10.96.0.1:443` or the API Service IP is unreachable, investigate service networking, kube-proxy, node routing, CNI dataplane state, and API server availability before blaming an application.

### Storage

For volume issues, follow the lifecycle:

1. PVC creation and binding to PV.
2. Provisioner/CSI controller events.
3. Scheduler volume constraints and topology.
4. Attach/detach operations.
5. Node CSI and kubelet mount logs.
6. Filesystem permissions, read-only mounts, capacity, inode exhaustion, and application write errors.

Separate storage control-plane failures from application-level database initialization or migration problems.

### Control Plane And API Extensions

For API server or aggregated API errors, identify whether the core API is affected or a single APIService is unavailable. `metrics.k8s.io` or metrics-server failures usually affect metrics/HPA visibility, not all workload traffic, unless they share a broader node/network problem.

For etcd, check leadership, quorum, slow requests, compaction, disk latency, database size, alarms, and apiserver client errors. Do not call etcd root cause from isolated watch or timeout messages unless the timing and scope match cluster-wide API symptoms.

### Rollout And Change Correlation

Treat Helm releases, image changes, config changes, node maintenance, certificate rotation, CNI changes, and cluster upgrades as timeline anchors. A change is causal only when the affected objects and symptom timing align and alternative explanations have been checked.

## Timeline Rules

Build a timeline from the earliest high-signal event through recovery or collection end.

- Normalize timestamps across pod logs, events, control-plane logs, system logs, and file metadata.
- Kubernetes component logs may use local time, UTC RFC3339, or `I0429 08:15:05` style without year. Infer the year only from file metadata or surrounding evidence and label it.
- Account for clock skew and collection-time artifacts.
- Put rollout events, pod restarts, node condition changes, DNS/network failures, storage events, and application errors on the same timeline.

## Evidence Standards

Every important claim should cite concrete evidence:

- File path and line number, or command section and source file.
- Concise excerpt or summary of the relevant lines.
- Why the evidence matters.

Use labels:

- **Fact**: directly observed in evidence.
- **Inference**: reasoned from one or more facts.
- **Unknown**: missing, ambiguous, or inconclusive evidence.

Avoid exposing secrets from manifests or logs. Redact tokens, passwords, private keys, certificates, image pull credentials, bearer tokens, kubeconfigs, cloud keys, and sensitive customer addresses when they are not necessary to the conclusion.

## Output

Lead with the answer, then show evidence. Prefer a few strong findings over a long list of weak clues.

Use this final shape unless the user requests otherwise:

1. **Executive Summary**: most likely issue, scope/impact, confidence.
2. **Current Hypotheses**: statuses and why each strengthened, weakened, or remains open.
3. **Strongest Evidence**: supporting and refuting evidence with paths/lines or command sections.
4. **Timeline**: ordered key events with timestamp assumptions.
5. **Conclusion**: confirmed root cause if justified; otherwise most likely cause and remaining uncertainty.
6. **Next Actions**: safest remediation or exact additional evidence needed.

If evidence is insufficient, say so early and specify the missing discriminator, such as `kubectl get events -A`, `kubectl get pods -A -o wide`, `kubectl describe pod`, previous container logs, Service/Endpoint YAML, node conditions, kubelet logs for the affected node, CNI logs, ingress controller logs, PVC/CSI events, Helm history, or manifests from before and after the change.

## Safety

Prefer offline analysis and read-only live checks. Do not recommend disruptive commands casually. Avoid suggesting pod deletion, rollout restarts, node drains, CNI restarts, control-plane restarts, storage detach/reattach, or configuration changes unless evidence supports them and safer checks are exhausted.
