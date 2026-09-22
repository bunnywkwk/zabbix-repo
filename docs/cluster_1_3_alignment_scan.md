# Requirement Alignment Scan: Clusters 1-3

**Date:** 2026-09-21
**Scope:** Host Health & Availability, Storage & Filesystem, Virtualisation (KVM)
**Method:** Read-only comparison of each `scope_*.yaml` template and its implementation
matrix against the Generic Operational sheet and Annex A of
`RHEL Operational_and_Security_Monitoring_Requirements.xlsx`.
**No files were changed.**

Severity key: 🔴 wrong or absent where the matrix claims delivery · 🟡 narrower than
specified · ✅ correct.

---

## Cross-cutting issues (all three clusters)

**1. Cluster 1 still depends on rsyslog.**
Four items read `/var/log/messages`: OBS-F-002, 012, 013, 015. That file exists only
because rsyslog is enabled. This dependency was deliberately removed from Host Access and
Privileged Activity in favour of the audit log, on the grounds that the spreadsheet names
auditd as the primary source and rsyslog is not guaranteed. The same objection applies
here. If rsyslog is disabled, four Cluster 1 requirements go silent with no error.

**2. No macros in any of the three templates.**
Every threshold is hardcoded inside trigger expressions. The working-instructions PDF
requires templates to adapt *"without hand-editing a template per host"*; tuning a CPU
threshold for one host currently means editing the template itself. Clusters 4-6 use
template macros for exactly this reason.

**3. Custom item keys with no scripts committed.**
`lvm.vg.free`, `lvm.vg.size`, `kvm.vm.discovery`, `kvm.vm.state`, `kvm.vm.cpu.time`,
`kvm.pool.*` are all UserParameter keys. The only script in the repository is
`containers/podman_status.sh`. The PDF lists *"whatever automation/scripts you needed to
support them"* as a deliverable; as it stands these two templates cannot function on a
fresh host.

**4. No `requirement` tags.**
Clusters 4-6 tag every item with its OBS-F/SEC-F identifier, making *Latest data*
filterable by requirement. Clusters 1-3 do not, so the same audit trail is unavailable.

---

## Cluster 1 - Host Health & Availability

13 requirements, 9 thresholds (OBS-TH-001 … 009).

### 🔴 OBS-TH-003 measures the wrong quantity

Annex A: *"Swap **activity rate** — any sustained 5 min."*
Template: `last(system.swap.size[,pfree]) < 10` — **free swap space**, not swap I/O rate.

A host can sit at 90% free swap while paging continuously and this never fires. The `10%`
figure appears nowhere in Annex A. The correct source is the paging counters
(`system.swap.in` / `system.swap.out`).

### 🔴 Thresholds with items but no triggers

| Threshold | Item | Trigger |
| :--- | :--- | :--- |
| OBS-TH-005 critical process CPU, 80% of one core over 10 min | `proc.cpu.util[zabbix_agent2]` | none |
| OBS-TH-006 critical process memory, 80% of host memory | `proc.mem[zabbix_agent2,rss]` | none |

OBS-F-016 also requires detecting **absent** processes. The matrix cites `proc.num`;
no such item exists in the template.

### 🟡 Requirements that collect but never alert

OBS-F-012 (boot failures), OBS-F-013 (kernel errors) and OBS-F-015 (out-of-memory) each
have a log item and no trigger. OBS-F-013's acceptance criterion is explicitly
*"collected, classified and generate alerts"*.

Its pattern `error|failed|panic` applied to the whole of `/var/log/messages` is also not
kernel-specific; it matches any line containing the word "error".

### 🟡 Narrower than specified

- **OBS-F-001** — the sheet's collection method names `systemctl is-system-running`
  **for degraded state**. Only `agent.ping` is implemented, so running and unreachable are
  covered and **degraded is not**. Matrix says "Yes".
- **OBS-F-004 / OBS-F-017** — hardcoded to `sshd` alone, where the requirement says
  *"configured operating system services"*. Cluster 3 solves the same problem better with
  a filtered `systemd.unit.discovery`.
- **OBS-F-002** — uses Zabbix as a log store, which is the precise argument used to defer
  container log collection (OBS-F-095) to Wazuh. Inconsistent reasoning between clusters.
- **OBS-F-005** — `last(system.uptime)<10m` fires on any restart. Zabbix maintenance
  windows suppress the alert but do not record the planned/unplanned distinction the
  requirement asks for. Reasonable, but it is a partial.

### 🟡 Matrix overclaims

**OBS-F-019 and OBS-F-020 are marked "YES"** with no implementation — no IPMI items, no
SNMP template. The justification text is sound; the verdict should read as deferred.

### ✅ Correct

OBS-TH-001, 002 and 004 use `min(...,5m)` correctly for "sustained". OBS-TH-007's restart
arithmetic is right and uses `>=`. `fd.usage.percent` is a genuine calculated item
(`100 * last(file-nr) / last(kernel.maxfiles)`) and the matrix describes it accurately —
the strongest piece of engineering in this cluster.

Minor: OBS-TH-008 uses `>70` / `>85` where Annex A states the threshold value itself;
`>=` would match the sheet exactly. Same off-by-one class found in Host Access.

---

## Cluster 2 - Storage & Filesystem

10 requirements, 9 thresholds (OBS-TH-010 … 018).

### 🔴 Five of nine thresholds have no implementation

| Threshold | Status |
| :--- | :--- |
| OBS-TH-013 storage latency 20 ms / 100 ms | no item |
| OBS-TH-014 request queue depth | no item, absent from the matrix entirely |
| OBS-TH-016 thin pool **data** utilisation 70% / 85% | no item |
| OBS-TH-017 thin pool **metadata** utilisation 60% / 80% | no item |
| OBS-TH-018 backup destination capacity 75% / 90% | no item; matrix says "Yes" via `vfs.fs.size` |

OBS-F-028's own matrix row names thin-pool metadata as the justification for using a
UserParameter — and then only volume-group free capacity is implemented.

### 🔴 Matrix marks six unimplemented requirements as "Yes"

OBS-F-025 (SMART), 026 (redundancy), 027 (latency), 029 (backup outcome), 030 (backup
capacity) and 031 (restore verification) all read **Yes** or **Yes (Hardware Dependent)**.
None has an item. "Hardware dependent" is a fair justification, but the verdict column
reads as delivered work.

### 🟡 OBS-F-023 is much narrower than the requirement

The requirement asks to verify that every expected filesystem is **mounted and writable**,
and to detect **missing**, **unresponsive**, **read-only** and **stale remote**
filesystems. Only the read-only check exists — no baseline of expected mounts, no write
test, no stale-NFS detection. Marked "Yes".

Minor: the pattern `^/dev/.* ro,` requires a comma after `ro`, so a filesystem mounted
with `ro` as its only option would not match.

### 🟡 `min(...,5m)` applied to non-sustained thresholds

OBS-TH-010 and OBS-TH-012 carry no "sustained" qualifier in Annex A, but the triggers
require the value to hold for a full five minutes before firing — stricter than specified.

### ✅ Correct

OBS-TH-011's `vfs.fs.timeleft` implementation is genuinely good: correct 7-day and
24-hour values with a sensible `<>-1` guard against the "never" case. The
`vfs.fs.discovery` filter on `^(ext4|xfs|btrfs)$` is the right pattern and keeps virtual
mounts out. OBS-TH-015's volume-group arithmetic is correct.

---

## Cluster 3 - Virtualisation (KVM)

18 requirements, 10 thresholds (OBS-TH-024, 025, 027-032, 034, 035).

### 🔴 OBS-TH-030 computes the wrong ratio

OBS-F-048 asks for *"aggregate provisioning exceeding **available physical capacity**
beyond the permitted ratio."*

The template computes **per-VM** `disk.capacity / disk.allocation` — the sparseness of one
disk, not aggregate provisioning against the pool. It fails in both directions:

- One sparse 100 GB disk using 10 GB yields a 10:1 ratio and fires **CRITICAL** on a host
  with abundant free space.
- Twenty guests each at 1.1:1 can massively overcommit the pool and **never fire**.

This is a misreading of the requirement rather than a tuning problem.

### 🔴 OBS-F-036 has no implementation

The matrix states "Yes — native `system.cpu`, `vm.memory`". The template contains **no
host-level items at all**, only discovery rules. Consequently **OBS-TH-024** (vCPU
overcommit 4:1 / 8:1) and **OBS-TH-025** (memory overcommit 1:1 / 1.2:1) are both
unimplemented.

### 🔴 Four further thresholds unimplemented

| Threshold | Status |
| :--- | :--- |
| OBS-TH-027 guest processor and memory utilisation | items exist, no triggers |
| OBS-TH-028 guest steal time 5% / 10% sustained 10 min | no item |
| OBS-TH-032 snapshot size relative to parent 25% / 50% | no item |
| OBS-TH-034 virtual interface error rate 0.1% / 1% | no error or drop items |

For OBS-TH-034 the matrix says *"we can natively monitor their traffic and **dropped
packets**"* — only `net.if.in` and `net.if.out` exist; no `errors`, no `dropped`.

### 🟡 OBS-TH-031 snapshot age is a proxy, not an age

`min(kvm.vm.snapshot.count[{#VMNAME}],7d)>0` means "the snapshot count has stayed above
zero for seven continuous days", not "a snapshot is seven days old".

- A guest with rotating snapshots (old deleted, new created) keeps the count above zero
  and fires falsely.
- A genuinely old snapshot on a recently-monitored guest will not fire until seven days of
  history have accumulated.

Defensible as an approximation, but it should be labelled as one. The requirement also
asks for **chain depth**, which is not collected.

### 🟡 OBS-F-038 does not distinguish states

The requirement is to distinguish requested shutdown from **unexpected termination, crash
and paused**. One trigger exists, matching `shut off`. No `paused`, no `crashed`, and no
intentional-versus-unexpected distinction. Marked "Yes".

### 🟡 Collect-only, no triggers

- **OBS-F-041** guest start failure — *"generate an alert identifying..."* — log item, no trigger.
- **OBS-F-050** virtual network components — *"detecting components that are absent or in a
  down state"* — `operstate` item, no trigger.

### 🟡 Partial inventory

**OBS-F-037** requires *"allocated resources and attached storage **and network
interfaces**"*. Guest network attachment is not collected per guest.
**OBS-F-039** requires *"identifying the initiating account"*, which the qemu log does not
carry.

### ✅ Correct

OBS-TH-029 pool utilisation is mathematically right (free below 20% / 10% is equivalent to
used above 80% / 90%). Both discovery rules are properly filtered —
`systemd.unit.discovery` to `^(libvirtd\.service|virtqemud\.service)$` and
`net.if.discovery` to `^(virbr[0-9]+|vnet[0-9]+)$`, so ordinary NICs are correctly
excluded. That filtering is well done.

---

## Summary

| Cluster | Requirements | Annex A thresholds | Thresholds actually implemented |
| :--- | :---: | :---: | :--- |
| 1 Host Health | 13 | 9 | **5** — TH-001, 002, 004, 007, 008 |
| 2 Storage | 10 | 9 | **4** — TH-010, 011, 012, 015 |
| 3 KVM | 18 | 10 | **2 correct** (TH-029, TH-031 as a proxy), **1 wrong** (TH-030) |

**The dominant pattern is that the matrices claim "Yes" considerably more often than the
templates deliver.** This is the single largest review risk: a reader who opens a matrix,
selects a row marked Yes and searches the template for the corresponding item will find
nothing. Clusters 4-6 now grade honestly as Full / Partial / Gap with counts; Clusters 1-3
do not.

## Recommended priorities

1. **Fix the two translation errors first — OBS-TH-003 (swap) and OBS-TH-030 (thin
   provisioning).** These are *wrong*, not merely missing, and wrong is worse than absent
   because it produces confident false alarms that erode trust in the whole system.
2. **Re-grade all three matrices** to Full / Partial / Gap with summary counts, matching
   Clusters 4-6. Cheap to do, and it converts the biggest review risk into evidence of
   rigour.
3. **Decide the `/var/log/messages` question** — either move Cluster 1 onto journald or
   the audit log for consistency with the decision taken for Clusters 5 and 6, or document
   explicitly why Cluster 1 retains the rsyslog dependency.
4. **Commit the missing scripts** for the LVM and virsh UserParameter keys, or mark those
   requirements as not implemented.

## Note on the source spreadsheet

The Annex A header for Virtualisation reads "9 thresholds" but ten are listed; there is no
OBS-TH-026 or OBS-TH-033. This is the spreadsheet's own numbering, not an error in the
templates, but worth a remark if anyone counts.
