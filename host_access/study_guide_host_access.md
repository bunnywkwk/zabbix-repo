# Study Guide: Host Access & Audit-Based Login Monitoring

This guide breaks down the engineering decisions made for the **Host Access**
cluster: recording logins, logouts, console access, and detecting brute force
against three different classes of account.

## Why This Cluster Reads the Audit Log

The first version tailed `/var/log/secure`, which required `rsyslog` to be
enabled so that `journald` would write a traditional flat file. That worked, but
it made the whole cluster depend on a service the requirements never asked for.

The spreadsheet names audit records as the primary source throughout:

| Requirement | Spreadsheet's stated collection method |
| :--- | :--- |
| SEC-F-001 | `ausearch -m USER_LOGIN`, successful results. *Also* `/var/log/secure`. |
| SEC-F-002 | `ausearch -m USER_LOGIN`, failed results. *Also* `lastb` and `/var/log/secure`. |
| SEC-F-004 | Audit `USER_END` records, logind session close, wtmp. |
| SEC-F-016 | Audit login records **filtered by terminal** for local and serial console. |

`auditd` receives events from the kernel over a netlink socket and writes its own
log with its own rotation — rsyslog is nowhere in that path, and auditd is
mandatory under CIS hardening, so on a compliant host the source is guaranteed
present.

Access is granted through auditd's own `log_group` directive rather than an ACL,
because auditd recreates its log on rotation and an ACL applied to the old file
would be silently lost. The full procedure is in the Privileged Activity study
guide, which configured it first; this cluster reuses the same access.

## One Record, Four Requirements

A single `USER_LOGIN` record carries everything this cluster needs:

```
type=USER_LOGIN msg=audit(1789956336.314:5197): pid=14369 uid=0 auid=4294967295
  ses=4294967295 subj=... msg='op=login acct="frqadmin"
  exe="/usr/libexec/openssh/sshd-session" hostname=? addr=127.0.0.1
  terminal=ssh res=failed'UID="root" AUID="unset"
```

| Field | Used for |
| :--- | :--- |
| `res=success` / `res=failed` | SEC-F-001 and SEC-F-002 |
| `acct=` | SEC-F-003 — which account class was targeted |
| `addr=` | source information required by SEC-F-001 |
| `terminal=` | SEC-F-016 — `ssh`, `tty1` (console), `ttyS0` (serial) |

This is the advantage of a structured record over a text line. `/var/log/secure`
expresses the same information in prose, so every field has to be recovered by
guessing at sentence structure that belongs to sshd rather than to a standard.

### The `terminal` field is what unlocked SEC-F-016

The previous matrix marked SEC-F-016 as entirely out of scope, on the grounds
that console access "bypasses the Linux OS Agent". That is true of the
**management controller** (iDRAC/iLO), which is a separate computer on a separate
network — but it is not true of the **physical or serial console**. Those produce
ordinary audit login records; they simply carry a different terminal.

So the requirement splits cleanly:

- **Physical console** (`terminal=tty1`) and **serial console** (`terminal=ttyS0`)
  are recorded by this template.
- **The management controller** genuinely requires IPMI/Redfish and remains out
  of scope.

That is a partial rather than a gap, and the distinction is worth making because
console logins are a real access path that was previously invisible.

## Counting Attempts, Not Connections

Annex B counts **attempts**: SEC-F-002 says "every failed user login attempt",
and SEC-F-003 alerts "when failed login attempts exceed a threshold". Choosing
the wrong audit record type silently changes what the thresholds mean.

A failed SSH authentication writes two different record types, and they do not
count the same thing. Measured on the target host by entering three wrong
passwords in a single `ssh` session:

| Record | Before | After | Delta |
| :--- | :--- | :--- | :--- |
| `USER_AUTH res=failed` | 13 | 16 | **+3** — one per password attempt |
| `USER_LOGIN res=failed` | 2 | 3 | **+1** — one per connection |

`USER_LOGIN` records the *outcome of the session*, not each attempt. Building
the thresholds on it would have undercounted by the number of retries sshd
allows — SEC-TH-002's critical value of 8 would have required roughly 24 real
attempts before firing.

So the failed-login items count `USER_AUTH`.

### Why they also filter on the login service

`USER_AUTH` is written by every PAM authentication, including **sudo**. An
unfiltered count would mix privilege elevation failures into a login brute force
count — and those already belong to the Privileged Activity cluster, so they
would be counted twice across two templates and neither number would mean what
it claims.

The items therefore match the login services (`sshd`, `/usr/bin/login`) and
exclude sudo. This is the mirror image of the Privileged Activity cluster, which
matches `USER_AUTH` *with* sudo for the same reason in reverse. Same record
type, opposite filter, because the requirements are asking different questions.

### The general lesson

Both of these were found by measuring rather than reasoning. The record type
that *sounds* right — `USER_LOGIN` for a login requirement — was the wrong one,
and a two-minute `grep -c` before and after a controlled test settled it. The
same approach corrected the `auid` assumption in SEC-F-019 and the Podman field
names in the Containers cluster.

## Three Account Classes, Three Thresholds

Annex B does not define one brute force threshold; it defines five, and they are
not interchangeable. Three are implemented here:

| Threshold | Account class | Warning | Critical | Window |
| :--- | :--- | :--- | :--- | :--- |
| SEC-TH-001 | Service / non-interactive | 3 | 5 | **10 min** |
| SEC-TH-002 | Standard interactive | 5 | 8 | 5 min |
| SEC-TH-003 | Privileged / administrative | 3 | 5 | 5 min |

The reasoning is in the spreadsheet's own rationale column. A service account
uses stored credentials and should essentially never fail, so its threshold is
tight and its window longer — a slow trickle of failures against a service
account is more suspicious than a burst against a human one. A standard user
mistypes passwords, so it gets the most tolerance. A privileged account has a
smaller population and a higher consequence, so it is tightened again.

Each class is a template macro, so a site with different account names changes a
macro rather than the template.

### Why the macros are prefixed `HA_`

Zabbix user macros resolve **per host**, not per template. The previous version
used `{$PRIV_ACCOUNT}`, which the Privileged Activity template also defines — with
a different meaning and a different value. Once both templates are linked to the
same host, one definition wins and the other cluster silently monitors the wrong
account. Prefixing avoids the collision entirely.

This is worth remembering generally: template macros share a namespace on the
host, so a macro name is effectively global.

### An off-by-one worth catching

The previous thresholds were written as `>5` and `>8`. Annex B states the
threshold **is** 5 and 8 — so `>5` first fires on the sixth attempt, and the
alarm was consistently one attempt late. All comparisons are now `>=`, matching
the spreadsheet exactly.

Small, but it is the kind of thing an auditor checks by reading the trigger
expression next to the threshold table.

## What Is Still Out, and Why It Is Not a Zabbix Failure

SEC-TH-004 (distributed guessing) and SEC-TH-005 (password spraying) count
failures **per source address**, not per account.

The data is there — every record carries `addr=`. The obstacle is structural.
Zabbix aggregates values belonging to a known item, and you cannot create an
item per attacking IP address because you do not know the address until the
attack happens. Discovery needs a bounded set to discover.

SEC-TH-005 is harder still: it requires counting *distinct accounts* per source
over 30 minutes, which is a set-cardinality operation over a correlated stream.

So the honest statement is not "this is inefficient in Zabbix" but "this is
event correlation over an unbounded key space, which is what a SIEM is for" —
and Wazuh is already deployed on every host in this estate.
