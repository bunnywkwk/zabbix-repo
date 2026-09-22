# Study Guide: Privileged Activity (Cluster 5)

This guide breaks down the engineering decisions made for the **Privileged
Activity** cluster: monitoring `sudo` usage, elevation failures and direct
privileged logins.

## Why This Cluster Reads the Audit Log, Not `/var/log/secure`

The first version of this template tailed `/var/log/secure`, the same file the
Host Access cluster uses. It worked, but it carried a hidden dependency: that
file only exists because `rsyslog` is enabled to receive a copy of the
authentication logs from `journald`. On a host where rsyslog is not wanted,
`/var/log/secure` is empty and the whole cluster goes silent.

Re-reading the requirements spreadsheet settles the question. The Collection
Method column for this cluster names the audit subsystem first every time:

| Requirement | Spreadsheet's stated collection method |
| :--- | :--- |
| SEC-F-005 | `ausearch -m USER_CMD` for sudo, `USER_START` for su. *Also* `/var/log/secure`. |
| SEC-F-018 | `ausearch -m USER_CMD`, failed results. Sudo denials *also* appear in `/var/log/secure`. |
| SEC-F-019 | Audit login records where the account is root and no originating login UID is set. |

`/var/log/secure` is the *alternative*, not the source of record. The Assumption
column for SEC-F-005 reads **"Audit subsystem enabled"** — it never mentions
rsyslog.

**`auditd` is genuinely independent.** The kernel audit subsystem sends events
over a netlink socket to the `auditd` daemon, which writes `audit.log` itself,
with its own rotation. `rsyslog` is nowhere in that path. Stop rsyslog and
`/var/log/secure` freezes while `audit.log` keeps recording — which is a
two-command demonstration:

```bash
sudo systemctl stop rsyslog
sudo true                                # generate an elevation event
sudo tail -3 /var/log/audit/audit.log    # still recording
sudo tail -3 /var/log/secure             # frozen
sudo systemctl start rsyslog
```

`auditd` is also mandatory under CIS hardening, so on a compliant host the
source is guaranteed present — a stronger assumption than rsyslog being enabled.

## Granting Access: `log_group`, Not `setfacl`

The Host Access cluster uses `setfacl -m u:zabbix:r /var/log/secure`. That
approach is **not** appropriate here, because `auditd` recreates its log file on
rotation and an ACL applied to the old file is silently lost — the item would
work for days and then quietly stop.

`auditd` provides the correct mechanism itself, in `/etc/audit/auditd.conf`:

```
log_group = zabbix
```

`auditd` then sets `0640 root:zabbix` on the log and **reapplies it on every
rotation**. The permission is owned by the daemon that owns the file.

One RHEL-specific trap: `systemctl restart auditd` is refused with *"unit may be
requested by dependency only"*. auditd must be restarted through the legacy
wrapper:

```bash
sudo service auditd restart
```

### Directory traversal, again

The same CIS traversal problem from the Host Access cluster applies, one level
deeper. `/var/log/audit` is mode `0700` or `0750` and owned by `root:root`, so
even with the file readable, the `zabbix` user cannot *enter* the directory to
reach it. An execute-only ACL on the directory is required:

```bash
sudo setfacl -m u:zabbix:x /var/log/audit
```

`x` without `r` lets the agent traverse to a known path but **not list** the
directory contents. Combined with `log_group` on the file itself, the agent can
open exactly one file in that directory and discover nothing else.

Verify as the agent's own account before touching the Zabbix GUI:

```bash
sudo -u zabbix head -1 /var/log/audit/audit.log
```

## Reading Audit Records

An audit record is one line of `key=value` pairs. A `sudo` execution looks like:

```
type=USER_CMD msg=audit(1789608414.562:498): pid=40069 uid=1000 auid=1000 ses=3
  subj=... msg='cwd="/home/bunny" cmd=646E6620696E7374616C6C202D79207A61626269782D6167656E7432
  exe="/usr/bin/sudo" terminal=pts/6 res=success'UID="bunny" AUID="bunny"
```

Three fields carry the security meaning:

- **`auid`** — the *audit login UID*. The account that originally logged in,
  preserved across every elevation. This is what makes attribution possible:
  `uid=0 auid=1000` is user 1000 acting as root, and the text log cannot express
  that distinction.
- **`res=success` / `res=failed`** — outcome of the operation.
- **`cmd=`** — the executed command.

### The hex encoding limitation

`cmd=` is **hex encoded whenever the command contains spaces**, and plain text
otherwise:

```
cmd="true"                                    <- single word, plain
cmd=646E6620696E7374616C6C...                 <- "dnf install -y zabbix-agent2"
```

auditd does this so that spaces and quotes inside a command cannot break the
record format. `ausearch -i` decodes it, but Zabbix's `log[]` item stores the
line verbatim. So SEC-F-005 records *that* a command ran, by *whom*, from
*where* — but the command string itself may be hex. This is documented as a
partial rather than papered over; decoding would need a JavaScript preprocessing
step, which is more machinery than this requirement justifies.

## Why SEC-F-018 Got More Accurate

The previous version matched sudo's English output: `incorrect password` or
`command not allowed`. That is brittle — the wording belongs to sudo, not to a
standard, and changes with locale and version.

Audit separates the two failure modes into distinct record types, both matched
by one item:

| Failure | Record |
| :--- | :--- |
| Wrong password at the sudo prompt | `type=USER_AUTH ... res=failed` |
| Command not permitted for the account | `type=USER_CMD ... res=failed` |

The `sum(10m)` trigger against SEC-TH-007 (3 warning / 5 critical) is unchanged
— only the source of the count improved.

## Why SEC-F-019 Got Better

Detecting direct root login from `/var/log/secure` meant matching
`Accepted.*for root`, which cannot distinguish a genuine direct login from
other appearances of the word.

Audit answers it structurally. A successful `USER_LOGIN` record carries the
account being logged into:

```
type=USER_LOGIN ... msg='op=login acct="root" exe="/usr/libexec/openssh/sshd-session"
  hostname=? addr=127.0.0.1 terminal=ssh res=success'UID="root" AUID="root"
```

The item matches `acct="<privileged account>"` together with `res=success`,
which is precisely the spreadsheet's stated method — *"login records where the
account is root"*. The account is parameterised through `{$PRIV_ACCOUNT}`, so a
site using a different privileged account changes only a macro.

**A note on `auid`.** An earlier draft keyed this on the audit login UID
(`auid=0`) instead. Testing on the target host showed why that was wrong: the
login UID is not assigned until the login succeeds, so a `USER_LOGIN` record
shows `auid=4294967295` / `AUID="unset"` at the moment of a failed attempt, and
its value at successful login could not be confirmed from the available records.
The account name is present in both cases and is what the requirement actually
names, so the item keys on that instead.

The dots surrounding the macro in the item key — `acct=.{$PRIV_ACCOUNT}.` —
match the double quotes auditd places around the account name. Writing literal
quotes inside a Zabbix item key requires nested escaping; a wildcard is cleaner
and equally precise here.

## Why SEC-F-006 Is Still Out

`USER_CMD`, `USER_AUTH` and `USER_LOGIN` are **single-line** records, which is
what makes native `log[]` collection viable for this cluster.

SEC-F-006 is different. Administrative command monitoring needs audit `execve`
rules, and one execve event is written as several linked records — `SYSCALL`,
`EXECVE`, `PATH`, `PROCTITLE` — sharing a single event ID. Zabbix's `log[]` item
reads one line at a time and has no way to reassemble them. `ausearch -k` and
Wazuh both do this natively.

So the gap is specific and honest: not *"Zabbix cannot read auditd"* — this
template proves it can — but *"Zabbix cannot correlate multi-record audit
events."*
