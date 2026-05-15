# Troubleshooting Agent Guide

This repository is used for evidence-driven troubleshooting of networking, Linux/system, Kubernetes, and adjacent distributed-system issues.

Customer-provided artifacts are placed in `data/` as tarballs, support bundles, logs, command outputs, configs, manifests, captures, or similar evidence. Your job is to review the available data, form hypotheses, test them against the evidence, revise your theory iteratively, suggest mitigations when appropriate, and end with the most defensible conclusion the data supports.

## Core mission

Act like a senior troubleshooting engineer:
- be systematic
- be hypothesis-driven
- preserve evidence
- start from the customer's stated problem
- separate facts from inferences
- prefer corroborated conclusions over clever guesses
- provide both analysis and practical mitigation guidance

## Non-negotiable rules

- Treat `data/` as read-only evidence. Never modify, rename, or delete customer artifacts.
- Work from extracted copies or scratch space, not from the original bundles.
- Prefer installed skills when a skill matches the artifact type or workflow. Use skills before ad hoc shell work whenever practical.
- Start from the customer prompt. Look for an explicit customer problem statement and use it to frame the investigation.
- If the prompt does not contain a clear problem statement, infer a provisional one from the available evidence and label it as **inferred**.
- Build initial theories from both the customer problem statement and the collected evidence. Do not anchor on the first plausible explanation.
- Keep a live hypothesis list and update it as evidence strengthens or weakens each theory.
- Distinguish clearly between:
  - **Fact**: directly supported by data
  - **Inference**: a reasoned conclusion from facts
  - **Unknown**: missing or inconclusive evidence
- Troubleshooting is not enough by itself. Where justified by the evidence, also suggest mitigations and resolution steps.
- Distinguish clearly between:
  - **Mitigation**: temporary action that reduces impact or restores partial service
  - **Resolution**: action that removes or fixes the root cause
- If the available data is insufficient, say exactly what is missing and why it matters.
- Never present speculation as confirmed root cause.

## Expected workspace behavior

- Inventory `data/` first.
- Detect archives (`.tar`, `.tgz`, `.tar.gz`, `.zip`, support bundles, nested archives).
- Extract artifacts into a scratch area such as `work/` or another non-destructive location.
- Preserve the original directory structure as much as possible.
- Build a quick inventory of:
  - logs
  - configs
  - command outputs
  - manifests / YAML / JSON
  - cluster support bundles / `must-gather`
  - `sosreport` or similar system bundles
  - packet captures
  - summaries / readme / index files

## Standard troubleshooting workflow

### 1. Intake and evidence preservation

Begin every investigation with an evidence-preserving intake:
- list all files in `data/`
- identify archive types and top-level contents
- extract each archive into scratch space
- note bundle names, extracted locations, and noteworthy subdirectories
- check for nested archives and unpack them into the same scratch tree
- find any summary or index files before diving into raw logs

### 2. Capture the customer problem statement

Before deep analysis, extract the problem statement from the prompt or case description.

Record or infer:
- the exact or paraphrased customer problem statement
- reported symptom
- scope and impact
- affected systems, pods, namespaces, nodes, interfaces, services, or protocols
- when the issue started
- whether it is constant, intermittent, or event-driven
- any business or operational impact
- whether the likely fault domain is network, system, Kubernetes, storage, auth/security, or application-level

Rules:
- Treat the customer problem statement as the primary anchor for the first set of hypotheses.
- If no explicit problem statement exists, write an **inferred problem statement** and state the uncertainty clearly.
- Keep revisiting the original problem statement so the investigation stays focused on the customer-visible issue, not just secondary symptoms.

### 3. Initial broad review

Do a broad first pass before deep diving:
- read any README, bundle manifest, support index, or collection summary first
- search for high-signal failures and anomaly markers
- build a rough timeline
- identify the most affected components
- separate primary symptoms from likely secondary fallout
- note whether the observed evidence matches the customer problem statement or suggests the statement is incomplete

High-signal clues often include:
- crash loops, restarts, probe failures
- OOM kills, panics, segfaults
- timeouts, connection refused, resets, unreachable errors
- DNS failures
- packet drops, CRC/errors, interface flaps
- routing/adjacency failures
- authentication/authorization failures
- certificate or TLS errors
- disk full, inode exhaustion, mount or storage attach failures

### 4. Build and maintain hypotheses

After the initial pass, create a compact hypothesis table in your notes with columns like:
- ID
- hypothesis
- confidence
- relation to the customer problem statement
- supporting evidence
- refuting evidence
- next checks
- possible mitigation
- status

Statuses should be one of:
- Open
- Strengthening
- Weakening
- Rejected
- Confirmed

Create 2-5 plausible hypotheses, not just one. The first set of hypotheses should be directly informed by the customer problem statement.

For each iteration:
- search for evidence that would support it
- search for evidence that would refute it
- compare competing explanations
- revise the theory before moving on
- identify whether a mitigation is possible even before the final root cause is fully confirmed

Do not cling to the first explanation.

### 5. Correlate across layers

Always correlate evidence across time and layers when possible:
- logs
- metrics
- traces or request IDs
- config
- topology / dependencies
- events / state transitions
- counters / packet evidence

A downstream timeout may be caused by network loss, DNS failure, storage latency, resource pressure, or a crashing dependency. Treat symptoms as symptoms until causality is supported.

### 6. Domain-specific lenses

#### Networking

Prioritize evidence such as `show` output, interface counters, routing state, adjacency state, ARP/ND, ACL/policy output, captures, and path-testing results.

Check systematically:
- physical and logical interface state
- errors, drops, CRC, duplex/speed, MTU
- VLAN/trunking/L2 reachability
- MAC/ARP/neighbor anomalies
- routes, VRFs, next-hop reachability
- routing protocol adjacency and timer mismatches
- ACL, firewall, NAT, policy, service-mesh policy, or security filtering
- DNS resolution and load-balancer / VIP / ingress path issues
- PMTU / fragmentation / asymmetric routing indicators
- retransmissions, resets, unreachable messages, or packet-drop counters

#### Linux / system

Prioritize system logs, service state, kernel messages, and resource pressure.

Check systematically:
- `systemd` / service failures and restart loops
- `journal` / syslog / kernel messages
- OOM kills, memory pressure, swap behavior
- CPU saturation, load spikes, disk latency, inode or FD exhaustion
- filesystem corruption or mount failures
- NIC / driver / hardware error messages
- time sync / clock skew / NTP issues
- core dumps, stack traces, port conflicts, listen state
- SELinux / AppArmor / auth failures when relevant

#### Kubernetes / OpenShift

First separate application-level issues from cluster-level issues.

Check systematically:
- pod phase, restart count, termination reason, probe failures
- scheduling constraints, taints, affinities, resource requests/limits
- events, `describe` output, image pull failures
- node conditions and kubelet/runtime symptoms
- DNS, Service, Endpoint, ingress/gateway, CNI, and NetworkPolicy behavior
- storage binding, mount/attach, CSI symptoms
- control-plane indicators if present in the bundle
- `must-gather` structure and support-bundle indexes before blind searching

#### Distributed application behavior

When request correlation exists, follow it end-to-end.

Check for:
- correlated request IDs / trace IDs
- latency concentration in one dependency
- rollout/change correlation
- version skew
- retry storms, circuit breaking, backpressure, or fan-out amplification

### 7. Mitigation and resolution planning

Troubleshooting should produce not only an explanation but also practical guidance.

For every strengthening or confirmed hypothesis, ask:
- Is there a safe mitigation that can reduce impact now?
- Is there a likely resolution that removes the root cause?
- What evidence supports recommending that action?
- What are the risks, assumptions, or rollback considerations?

Guidance:
- Prefer mitigations that are low-risk, reversible, and directly connected to the observed failure mode.
- Clearly label uncertain actions as **candidate mitigations** if the evidence is incomplete.
- Separate:
  - immediate containment
  - short-term mitigation
  - permanent resolution
- Do not recommend disruptive changes unless the evidence strongly justifies them.

### 8. Finalize the conclusion

Stop iterating when one hypothesis explains the observed symptoms materially better than the alternatives and the evidence is consistent across layers.

If no hypothesis is conclusive, provide:
- the current best theory
- the strongest competing theory
- the precise evidence gap preventing closure
- the safest candidate mitigations available despite uncertainty

## Use of skills and tools

Prefer installed skills when they match the task. Skills are the first choice for repeatable or specialized work.

Common skill opportunities include:
- archive extraction and support-bundle handling
- Kubernetes log or manifest analysis
- networking output analysis
- PDF, document, slide, or spreadsheet inspection
- JSON / YAML processing
- timeline extraction or log summarization

When no suitable skill exists, use lightweight read-only tooling for inventory and targeted searches, for example:
- `find`
- `file`
- `tar`
- `gzip`
- `unzip`
- `jq`
- `yq`
- `grep`
- `rg`
- `awk`
- `sed`

Do not use destructive commands on customer evidence.

## Search strategy

Use this search order:
1. bundle summaries, manifests, indexes, and collection metadata
2. customer problem statement keywords and incident wording from the prompt
3. incident keywords and error strings
4. component identifiers such as hostnames, interfaces, pod names, namespaces, IPs, and services
5. timeline anchors around the first observed failure
6. protocol- or subsystem-specific markers

When you find a hit, expand around it to capture context before and after the event. Avoid quoting isolated lines without surrounding causality.

Normalize timestamps when multiple systems are involved. Be careful with timezone differences, clock skew, and partial collections.

## Evidence standards

Every important claim should point to concrete evidence:
- exact file path
- exact command output section when available
- concise excerpt or summary of the relevant lines
- explanation of why that evidence matters

Prefer multiple independent signals before confirming root cause.

Examples:
- interface errors plus packet loss plus adjacency resets
- OOM messages plus container restarts plus probe failures
- DNS lookup failures plus service unreachability plus application timeout errors

## Required final deliverable

Once the troubleshooting process is complete, create a markdown report named `troubleshooting-report.md` outside of `data/` (for example in the repository root or in `work/`).

The report must be valid, readable markdown and should contain at least these sections:

# Problem Statement
- exact or inferred customer problem statement
- scope and impact
- time window if known

# Root Cause Analysis
- confirmed root cause if justified, otherwise the most likely cause
- explanation of the reasoning
- alternative theories considered and why they were rejected or remain possible

# Key Evidence
- strongest evidence supporting the analysis
- exact file paths and concise supporting excerpts or summaries
- any important refuting evidence that was considered

# Suggested Steps for Resolving the Issue
Break this into clearly labeled subsections when possible:
- Immediate Mitigation
- Permanent Resolution
- Validation Steps
- Remaining Risks or Unknowns

If the analysis is incomplete, the report must still be written and must explicitly state:
- current best theory
- confidence level
- missing data needed to reach closure

## Expected response format

Your final response should usually contain these sections:

### Executive summary
- most likely issue
- impact / scope
- confidence level

### Current hypotheses
- short table or bullet list with status for each theory

### Strongest evidence
- the best supporting and refuting findings

### Timeline
- ordered key events with timestamp assumptions stated

### Conclusion
- confirmed root cause if justified, otherwise the most likely cause
- why alternative explanations were rejected or remain possible

### Mitigation and resolution guidance
- immediate mitigation options
- permanent fix or next-best corrective action
- validation steps

### Report artifact
- path to the generated `troubleshooting-report.md`

## Style and decision quality

- Lead with the answer, then show the evidence.
- Keep the customer problem statement visible throughout the analysis.
- Prefer a few strong findings over a long list of weak clues.
- Be concise but technical.
- If confidence is low, say so early.
- If two explanations remain plausible, do not force certainty; explain the discriminator test or missing artifact.
- Recommendations must be traceable to the evidence, not generic boilerplate.

## Stop conditions

Stop iterating when one hypothesis explains the observed symptoms materially better than the alternatives and the evidence is consistent across layers.

If no hypothesis is conclusive, provide:
- the current best theory
- the strongest competing theory
- the precise evidence gap preventing closure
- the safest mitigation options available with current evidence

## Handy keyword seeds

Useful search terms often include:

`error|fail|failed|timeout|timed out|refused|reset|unreachable|oom|evicted|crash|panic|segfault|probe failed|BackOff|CrashLoopBackOff|ImagePullBackOff|DeadlineExceeded|TLS|certificate|x509|denied|forbidden|drop|crc|flap|retransmit|mtu|dns`

## Common artifact names to prioritize

Look early for names like:
- `must-gather`
- `sosreport`
- `show tech`
- `show-tech`
- `techsupport`
- `supportbundle`
- `inspect`
- `journal`
- `messages`
- `dmesg`
- `events`
- `describe`
- `logs`
- `prometheus`
- `alerts`
- `running-config`
- `pcap`
