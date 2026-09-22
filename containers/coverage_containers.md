# Coverage & Test Tracker: Containers (Cluster 4)

Source of truth: `RHEL Operational_and_Security_Monitoring_Requirements.xlsx`,
Containers cluster = **12 requirements (OBS-F-084 … OBS-F-095)** and
**6 thresholds (OBS-TH-036 … OBS-TH-041)**.

In Zabbix, filter *Latest data* by tag `requirement` to see the items behind
each row below.

## Why the item count is large

Low-Level Discovery instantiates every prototype once per container:

```
8 host items + (9 prototypes x N containers)   = items
1 host trigger + (8 prototypes x N containers) = triggers
```

With 4 containers that is 44 items and 33 triggers from 7 requirements.
A single requirement often needs several items: OBS-F-088 needs `exited`,
`exitcode` and `startedat` because its trigger logic spans all three.

## Requirement coverage

| ID | Requirement | State | Items / triggers | Tested |
| :--- | :--- | :--- | :--- | :---: |
| OBS-F-084 | Runtime service monitoring | Full | `podman.present`, `podman.runtime_ok` + host trigger | [ ] |
| OBS-F-085 | Runtime config conformance | Out | — | n/a |
| OBS-F-086 | Container inventory | Partial | `containers.total`, `state`, `image` | [ ] |
| OBS-F-087 | Lifecycle event recording | Out | — | n/a |
| OBS-F-088 | Restart loop & abnormal exit | Full | `exited`, `exitcode`, `startedat` + 3 triggers | [ ] |
| OBS-F-089 | Resource utilisation | Partial | `cpu`, `mem` + 4 triggers | [ ] |
| OBS-F-090 | Resource limit breach | Partial | `exitcode` + OOM trigger | [ ] |
| OBS-F-091 | Image inventory & drift | Out | — | n/a |
| OBS-F-092 | Image provenance | Out | — | n/a |
| OBS-F-093 | Storage consumption | Partial | `storage.images`, `.containers`, `.volumes`, `.reclaimable` | [ ] |
| OBS-F-094 | Network attachment | Partial | `networks`, `ports` | [ ] |
| OBS-F-095 | Container log collection | Out | — | n/a |

**2 full, 5 partial, 5 out of scope.**

### What is missing from each partial

- **OBS-F-086** — mounted storage and allocated resource limits. `Mounts` is
  present in `podman ps` output and can be added; limits are not exposed there.
- **OBS-F-089** — network and block I/O. Both are in `podman stats` output
  (`net_io`, `block_io`) and can be added.
- **OBS-F-090** — CPU throttling (OBS-TH-038) and detection of containers with
  no limits defined.
- **OBS-F-093** — consumption is collected in bytes but not as a percentage,
  so OBS-TH-040 has no trigger. Orphan age (OBS-TH-041) is not exposed by
  Podman; only reclaimable bytes are.
- **OBS-F-094** — actual attachment is collected; conformance comparison needs
  an approved network baseline, which the project has not defined.

## Threshold coverage

| Threshold | Requirement | Value | State | Tested |
| :--- | :--- | :--- | :--- | :---: |
| OBS-TH-036 | OBS-F-088 | 3 / 5 restarts in 10 min | Met exactly | [ ] |
| OBS-TH-037 | OBS-F-089 | 80% / 95% of limit | Memory exact; CPU measured as percent of one core | [ ] |
| OBS-TH-038 | OBS-F-090 | Throttling any in 5 min | Not implemented | n/a |
| OBS-TH-039 | OBS-F-090 | Memory kill, any occurrence | Met | [ ] |
| OBS-TH-040 | OBS-F-093 | 75% / 90% | Bytes collected, no percentage trigger | n/a |
| OBS-TH-041 | OBS-F-093 | Orphan age 30 days | Not exposed by Podman | n/a |

## Test procedure

| # | Requirement | Action | Expected |
| :--- | :--- | :--- | :--- |
| 1 | OBS-F-089 | No action; `heavy-workload` sits at 99.96% of its 256m limit | Memory HIGH trigger fires |
| 2 | OBS-F-088 | `sudo podman stop -t 0 <name>` | Abnormal exit WARNING fires |
| 3 | OBS-F-090 | Run a container with `--memory=128m` and `stress-ng --vm-bytes 512M` | Exit code 137, OOM HIGH fires |
| 4 | OBS-F-088 | Start `crashloop.service` (systemd supervised) | `startedat` changes; WARNING at 3, HIGH at 5 within 10 min |
| 5 | OBS-F-084 | `sudo systemctl stop podman.socket` is not sufficient; simulate by making the script's podman call fail | Runtime HIGH fires |
| 6 | OBS-F-086/093/094 | None; verify values present in Latest data | Inventory, storage and network items populated |

## Notes

- Podman is daemonless. Its `Restarts` counter reflects only Podman's own
  restart policy, which does not run without a supervising systemd unit, and
  stayed at 0 through real crashes on this host. Restart detection therefore
  keys on `StartedAt`, which changes whoever performed the start.
- Exit code 137 is SIGKILL, the signal the kernel OOM killer sends. It is used
  as the memory-kill signal; this was independently confirmed against
  `podman inspect .State.OOMKilled` during earlier testing.
