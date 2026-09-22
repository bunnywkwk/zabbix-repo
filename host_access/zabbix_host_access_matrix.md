# Implementation Matrix: Host Access (Cluster 6)

This matrix evaluates the security requirements for the **Host Access** cluster,
aligning with the architect's PDF instructions to prefer native checks, avoid
unnecessary root privileges, and document hard gaps.

## Log source decision

This cluster reads **`/var/log/audit/audit.log`**, not `/var/log/secure`.

The requirements spreadsheet names audit records as the primary collection
method throughout — `ausearch -m USER_LOGIN` for SEC-F-001 and SEC-F-002, audit
`USER_END` records for SEC-F-004, and audit login records filtered by terminal
for SEC-F-016 — with `/var/log/secure` listed only as a secondary "also".

Reading audit directly removes the dependency on rsyslog being enabled. It also
improves three requirements, noted in the table below. The rsyslog-based version
is retained as `scope_host_access_rsyslog.yaml`.

| ID | Title | Threshold (Annex B) | Approach | Achievable? & Justification |
| :--- | :--- | :--- | :--- | :--- |
| **SEC-F-001** | Successful Login Monitoring | None | Native Check (`log`) | **Yes.** `USER_LOGIN ... res=success` captures every successful login with account, source address, hostname and terminal. Matching the audit record rather than sshd text also catches authentication methods the previous `Accepted publickey\|Accepted password` regex missed, such as keyboard-interactive and gssapi. |
| **SEC-F-002** | Failed Login Monitoring | None | Native Check (`log.count`) | **Yes.** A single item counts `USER_LOGIN ... res=failed` for **all** accounts, as the requirement specifies. The per-account items exist to drive SEC-F-003's thresholds and do not replace it. |
| **SEC-F-003** | Brute Force Detection | SEC-TH-001 … SEC-TH-005 | Native Trigger (`sum()`) | **Partial.** Three account classes implemented against their exact Annex B values: SEC-TH-001 service (3/5 in 10 min), SEC-TH-002 standard (5/8 in 5 min), SEC-TH-003 privileged (3/5 in 5 min). *Gap:* SEC-TH-004 and SEC-TH-005 — see the note below. |
| **SEC-F-004** | Logout Monitoring | None | Native Check (`log`) | **Yes.** `USER_END` records session close structurally, rather than matching the English phrase "session closed for user". |
| **SEC-F-016** | Console & Out-of-Band Access | None | Native Check (`log`) | **Partial.** The audit `terminal` field distinguishes the access path: `ssh` for network, `tty1..N` for the physical console, `ttyS0` for serial. Console and serial console access are therefore recorded natively. *Gap:* the platform management controller (iDRAC/iLO) bypasses the operating system entirely and requires an IPMI/Redfish template pointed at the management network — out of scope for an OS-level template. |

## Summary

| Outcome | Count | IDs |
| :--- | :--- | :--- |
| Fully implemented | 3 | 001, 002, 004 |
| Partially implemented | 2 | 003, 016 |
| Documented gap | 0 | — |

## Note on SEC-TH-004 and SEC-TH-005

These two thresholds count failures **per source address** rather than per
account — distributed guessing (10/20 from one source in 5 min) and password
spraying (5/10 distinct accounts from one source in 30 min).

The data is present: every `USER_LOGIN` record carries `addr=`. The limitation
is structural rather than a matter of efficiency. Zabbix counts values belonging
to a known item, and an attacking source address is not known in advance, so
there is no bounded key to build a discovery rule around. Grouping an unbounded
set of source addresses and correlating distinct accounts per source is
event-correlation work, which is what a SIEM does and what Wazuh is already
deployed to do on these hosts.

## Correctness changes from the previous version

- Thresholds used `>` where Annex B states the threshold value itself, so alerts
  fired one attempt late (`>5` fires at 6). All comparisons are now `>=`.
- Account macros are prefixed `HA_`. The previous `{$PRIV_ACCOUNT}` collided
  with the macro of the same name in the Privileged Activity template; macros
  resolve per host, so linking both templates made one of them incorrect.
- SEC-F-002 previously counted only two named accounts, leaving failures against
  any other account unrecorded despite the requirement stating "every failed
  user login attempt".
