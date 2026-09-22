# Implementation Matrix: Privileged Activity (Cluster 5)

This matrix evaluates the security requirements for the **Privileged Activity**
cluster. Collection is native Zabbix flat-file log monitoring against the Linux
audit log, with complex multi-record `auditd` correlation explicitly deferred to
Wazuh.

## Log source decision

This cluster reads **`/var/log/audit/audit.log`**, not `/var/log/secure`.

The requirements spreadsheet names the audit subsystem as the primary
collection method for every requirement in this cluster — `ausearch -m USER_CMD`
for SEC-F-005 and SEC-F-018, audit login records for SEC-F-019 — and lists
`/var/log/secure` only as a secondary "also". The Assumption column for
SEC-F-005 reads *"Audit subsystem enabled"*, not "rsyslog enabled".

Reading the audit log directly removes the rsyslog dependency entirely. `auditd`
receives events from the kernel audit netlink socket and writes its own log, so
it continues to record privilege elevation with rsyslog stopped or uninstalled.
The rsyslog-based version of this template is retained as
`scope_privileged_activity_rsyslog.yaml` for comparison.

| ID | Title | Threshold (Annex B) | Approach | Achievable? & Justification |
| :--- | :--- | :--- | :--- | :--- |
| **SEC-F-005** | Privilege Escalation Monitoring | None | Native Check (`log`) | **Partial.** Zabbix natively tails `audit.log` for `USER_CMD ... res=success`, capturing the invoking account, working directory and terminal for every `sudo` execution. *Limitation:* the `cmd=` field is hex encoded whenever the command contains spaces, so the executed command is recorded verbatim but not decoded. `ausearch -i` decodes it for investigation; Zabbix does not. |
| **SEC-F-018** | Failed Privilege Escalation | SEC-TH-007 (3/5 in 10m) | Native Trigger (`sum()`) | **Yes.** A single item counts both elevation failure modes — `USER_AUTH res=failed` for a wrong sudo password and `USER_CMD res=failed` for a command not permitted — and the `sum(10m)` trigger matches the Annex B threshold exactly. This is more precise than the previous text match, which relied on sudo's English message wording. |
| **SEC-F-019** | Direct Privileged Login Detection | Alert on Occurrence | Native Check & Trigger | **Yes.** Detected from the `acct=` field of a successful `USER_LOGIN` record, which is the spreadsheet's stated method — "login records where the account is root". More precise than matching an sshd text line, because the audit record names the account structurally rather than in prose. Keying on the audit login UID was tested and rejected: `auid` is unassigned at the point a login is recorded. |
| **SEC-F-006** | Administrative Command Monitoring | None | Deferred (Wazuh) | **No.** Requires audit `execve` rules retrieved with `ausearch -k`. Unlike the `USER_*` records above, an execve event is written as several linked records (`SYSCALL`, `EXECVE`, `PATH`, `PROCTITLE`) sharing one event ID. Zabbix's `log[]` item is line-oriented and cannot reassemble them. Wazuh performs this correlation natively. |

## Summary

| Outcome | Count | IDs |
| :--- | :--- | :--- |
| Fully implemented | 2 | 018, 019 |
| Partially implemented | 1 | 005 |
| Documented gap | 1 | 006 |

## Note on record structure

The `USER_CMD`, `USER_AUTH` and `USER_LOGIN` records this cluster depends on
are **single-line**, which is what makes native `log[]` collection viable. Only
syscall-derived events such as `execve` are multi-record, which is why
SEC-F-006 specifically remains out of scope rather than the cluster as a whole.
