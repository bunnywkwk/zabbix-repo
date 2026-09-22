# Implementation Matrix: Containers (Cluster 4)

This matrix evaluates all **12 requirements** of the Containers cluster
(`OBS-F-084` … `OBS-F-095`) against the architect's PDF instructions: prefer
native checks, avoid unnecessary root/sudo, avoid unmaintained third-party
code, and document honestly anything that cannot be turned into a real Zabbix
check.

## Collection decision

The Containers cluster is collected through **one UserParameter
(`podman.status`) backed by a read-only Bash script**, with every other item
derived from its output as a Zabbix dependent item.

Native container keys were evaluated first and are **not available on this
platform**:

| Check | Result |
| :--- | :--- |
| `zabbix_agent2 -p` | No `docker.*` or `podman.*` keys present in the agent binary |
| `dnf search zabbix-agent2-plugin` | Returns ceph, ember-plus, mongodb, mssql, nvidia-gpu, postgresql only — no container plugin |
| Agent version | `zabbix-agent2-8.0.0-beta2.release1.el9` |

Installing Docker was rejected: RHEL 9 does not ship it, it would require a
third-party vendor repository on a CIS-hardened host, it reintroduces a
root-owned daemon whose socket is root-equivalent, and the Zabbix agent still
has no plugin to talk to it. Aliasing `docker` to `podman` would not help
either, because the Zabbix Docker plugin speaks the API socket and never calls
the CLI.

## Requirement coverage

| ID | Title | Threshold (Annex A) | Approach | Achievable? & Justification |
| :--- | :--- | :--- | :--- | :--- |
| **OBS-F-084** | Container Runtime Service Monitoring | None | Dependent item (`podman.runtime_ok`) | **Yes.** The script's `podman ps` call doubles as the liveness probe: if the runtime does not service a container management operation, `runtime_ok` reports 0 and the host trigger fires. This tests the property the requirement actually names, rather than only the socket unit state. |
| **OBS-F-085** | Runtime Configuration Conformance | None | Deferred | **No.** The requirement compares runtime and per-container configuration against an *approved baseline*. The spreadsheet's own Assumption column records that baseline as "to be defined". With no baseline in existence there is nothing to compare against in any tool. Revisit once the project defines one. |
| **OBS-F-086** | Container Inventory Monitoring | None | Dependent LLD + items | **Partial.** Image reference, network attachment, lifecycle state and host container count are collected for every container. *Gap:* mounted storage and allocated resource limits — `Mounts` is present in `podman ps` output and can be added; the resource limits are not exposed there and would need a per-container `podman inspect`, which is exactly the per-container call this design removed. |
| **OBS-F-087** | Container Lifecycle Event Recording | None | Deferred | **No.** `podman events` is a continuous event stream, not a pollable value. Capturing it requires a persistent tailing process, which Zabbix's polling agent is not. Container lifecycle records reach journald and belong to the log platform (Wazuh), consistent with OBS-F-095. |
| **OBS-F-088** | Restart Loop and Abnormal Exit Detection | `OBS-TH-036` (3 / 5 in 10 min) | Dependent items + `changecount()` | **Yes.** Abnormal exit uses `Exited` and `ExitCode`. Restart rate uses `changecount()` over the container's `StartedAt` timestamp. See the study guide for why `StartedAt` is used rather than Podman's own `Restarts` counter. |
| **OBS-F-089** | Container Resource Utilisation | `OBS-TH-037` (80% / 95%) | Dependent items from `podman stats` | **Partial.** Memory is measured exactly against the container's configured limit, satisfying OBS-TH-037 precisely. CPU is reported by Podman as a percentage of one core, not of the container's quota, so the CPU threshold is expressed through template macros and documented as such. *Gap:* per-container network and block I/O — both are in `podman stats` output and can be added. |
| **OBS-F-090** | Container Resource Limit Breach | `OBS-TH-038`, `OBS-TH-039` | Dependent item on `ExitCode` | **Partial.** Memory-limit termination (`OBS-TH-039`, any occurrence) is detected via exit code 137, the SIGKILL the kernel OOM killer sends; this was cross-checked against `podman inspect .State.OOMKilled` during testing. *Gap:* CPU throttling (`OBS-TH-038`) requires reading per-container cgroup counters under `/sys/fs/cgroup`, and detection of containers with no limits defined requires per-container inspection. Neither justifies the added collection cost at this stage. |
| **OBS-F-091** | Container Image Inventory and Drift | None | Deferred | **No.** Drift is defined against an *approved image baseline* which the spreadsheet records as "to be defined and maintained by the project". Comparing running digests against a corporate baseline is configuration management (Ansible) or a security scanner (Wazuh/Trivy), not an infrastructure monitoring metric. |
| **OBS-F-092** | Container Image Provenance Verification | None | Deferred | **No.** Signature and provenance verification is enforced by the runtime's trust policy and recorded at image pull or container start. Zabbix has no reliable, safe way to re-verify cryptographic signatures, and doing so would duplicate a control that belongs in the CI/CD pipeline or the signing platform. |
| **OBS-F-093** | Container Storage Consumption | `OBS-TH-040`, `OBS-TH-041` | Dependent items from `podman system df` | **Partial.** Consumption by images, writable layers and volumes is collected in bytes, plus total reclaimable bytes as a proxy for unassociated items. *Gap:* `OBS-TH-040` is a percentage threshold and no percentage item exists yet — this is closable natively with a calculated item against `vfs.fs.size`. `OBS-TH-041` (orphan age 30 days) is a genuine data-source gap: `podman system df` reports reclaimable size but not the age of orphaned images or volumes. |
| **OBS-F-094** | Container Network Attachment | None | Dependent items | **Partial.** Actual network attachment and published host ports are collected per container. *Gap:* the conformance comparison, which again depends on an approved network baseline the project has not defined. |
| **OBS-F-095** | Container Log Collection | None | Deferred | **No.** Zabbix is not a log aggregation platform. Container output reaches the host through the journald log driver and is forwarded by the central logging solution (Wazuh), which is where the requirement's own Collection Method column points. |

## Summary

| Outcome | Count | IDs |
| :--- | :--- | :--- |
| Fully implemented | 2 | 084, 088 |
| Partially implemented | 5 | 086, 089, 090, 093, 094 |
| Documented gap | 5 | 085, 087, 091, 092, 095 |

Of the five documented gaps, **four (085, 091, 092, 094-conformance) are
blocked on baselines the project has not yet defined**, not on a limitation of
Zabbix. They become implementable the moment those baselines exist.

Threshold coverage and the per-requirement test procedure are tracked in
`coverage_containers.md`.
