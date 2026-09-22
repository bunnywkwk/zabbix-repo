# Study Guide: Containers & Podman Monitoring

This guide breaks down the engineering decisions made for the **Containers**
cluster: why the native route was closed, what was rejected and why, and the
exact procedure followed on `rhel9-hardened`.

## The Problem: There Is No Native Container Plugin

Every other cluster in this project uses native Zabbix agent keys. Containers
cannot, and it is worth proving that rather than assuming it.

Zabbix Agent 2 normally ships a Docker plugin (`docker.container.discovery`,
`docker.container.stats`) which can also monitor Podman, because Podman serves
a Docker-compatible REST API. On this platform it is simply absent:

```bash
zabbix_agent2 -p | grep -i -E 'docker|podman'    # no docker.* keys returned
dnf search zabbix-agent2-plugin                  # ceph, ember-plus, mongodb,
                                                 # mssql, nvidia-gpu, postgresql
```

The agent rejects the key with `ZBX_NOTSUPPORTED: Unknown metric
docker.container.discovery`. That specific wording matters: **"Unknown metric"
means the plugin is not in the binary at all.** A plugin that existed but could
not reach the socket would return a connection error instead. This is the
evidence that closed the question.

## The Unsafe Methods (What your mentor wants to avoid)

Four routes were considered and rejected before settling on the final design.

1. **Install Docker and alias `podman` to it.** RHEL 9 does not ship Docker;
   Red Hat removed it in favour of Podman. This would mean adding a
   third-party vendor repository to a CIS-hardened host — a far larger
   compliance deviation than a small auditable script. It also reintroduces a
   **root-owned daemon**, and membership of the `docker` group is effectively
   root on the host. *And it would not even work:* the Zabbix plugin talks to
   an API socket, not the CLI, so a `docker → podman` symlink changes nothing.

2. **Download the missing plugin binary from the internet.** Directly against
   the architect's instruction to avoid unmaintained third-party code, and
   against CIS package-provenance policy.

3. **Grant the agent broad `sudo podman` rights.** The obvious shortcut, and
   the one that quietly hands the monitoring account the ability to create,
   delete and modify containers — including mounting the host filesystem into
   a privileged container.

4. **Give the agent access to the Podman API socket.** This looks like the
   "clean native" answer, but it is worth understanding why it is *not* the
   least-privilege answer. **Access to `/run/podman/podman.sock` is full
   container control** — create, delete, mount host paths. Compared against a
   sudoers rule limited to three read-only commands, the socket is the larger
   privilege grant, not the smaller one.

## The Chosen Method: One Master Item, Surgical Sudo

The design collects the entire cluster with **a single script invocation per
minute**, no matter how many containers exist on the host.

**1. A read-only script** at `/etc/zabbix/scripts/podman_status.sh` runs three
commands and returns one JSON document describing the whole host — container
inventory, per-container statistics, and storage consumption.

**2. A surgical sudoers rule** at `/etc/sudoers.d/zabbix_podman`:

```
zabbix ALL=(root) NOPASSWD: /usr/bin/podman ps -a --format json, /usr/bin/podman stats --no-stream --format json, /usr/bin/podman system df --format json
```

Three exact commands. **No wildcards.** This is stricter than the common
pattern of `podman inspect *`, which accepts any argument. The Zabbix agent
physically cannot start, stop, remove or modify a container, an image or a
volume — it can only read state.

**3. One UserParameter** at `/etc/zabbix/zabbix_agent2.d/podman.conf`:

```
UserParameter=podman.status,/etc/zabbix/scripts/podman_status.sh
```

**4. Everything else is a dependent item.** The template extracts each value
from that one JSON document using JSONPath preprocessing. Dependent items never
touch the monitored host — they are pure server-side data shaping. Container
discovery is itself a dependent LLD rule, so even discovery costs no extra
call.

**Why this satisfies the mentor's criteria:** the host is queried once per
minute regardless of container count, the privilege granted is three read-only
commands, and the whole collection surface is a dozen lines of shell that an
auditor can read in one sitting.

## The CIS Permission Trap (Procedure)

On a hardened RHEL host, files created by `root` are unreadable by the `zabbix`
service account, and the agent crashes on startup rather than skipping the
file. This was encountered twice during deployment. The permissions are part
of the procedure, not a troubleshooting step:

```bash
sudo chown root:zabbix /etc/zabbix/zabbix_agent2.d/podman.conf
sudo chmod 640        /etc/zabbix/zabbix_agent2.d/podman.conf
sudo chown -R root:zabbix /etc/zabbix/scripts
sudo chmod 750           /etc/zabbix/scripts /etc/zabbix/scripts/podman_status.sh
sudo chown root:root  /etc/sudoers.d/zabbix_podman
sudo chmod 440        /etc/sudoers.d/zabbix_podman
```

`root` keeps full control, the `zabbix` group gets read and execute, everyone
else is blocked. Always validate the sudoers file with `visudo -c` before
restarting the agent.

**Verify as the agent's own user before touching the Zabbix GUI.** This single
command catches permission, sudoers and JSON problems in one step, instead of
waiting sixty seconds for a red error in Latest Data:

```bash
sudo -u zabbix /etc/zabbix/scripts/podman_status.sh | head -c 400
```

A correct result begins `{"present":1,"runtime_ok":1,"containers":[...`.

## Finding: Podman Is Daemonless, So `Restarts` Stays at Zero

This is the most important technical finding of the cluster, and a genuine
difference between Podman and Docker.

A container was deployed with `--restart=always`, crashed, and remained dead.
Two days later `podman ps` still reported:

```json
"Restarts": 0,  "State": "exited",  "ExitCode": 2
```

**Podman has no background daemon.** Its restart policy is enforced by a
supervising process, and without one — typically a systemd unit or Quadlet —
nothing restarts the container and the internal counter never increments. On
RHEL 9, production containers are systemd-managed for exactly this reason.

An earlier draft of this cluster built the restart trigger on that counter,
which meant the alarm could only be demonstrated by hard-coding a fake value.
That is not evidence.

**The fix:** key restart detection on the container's `StartedAt` timestamp and
count how many times it changes:

```
changecount(/Scope Containers/podman.container.startedat["{#NAME}"],10m)>=3
```

`StartedAt` changes on every genuine start, whoever performed it — Podman's own
policy, a systemd unit, or an administrator. The trigger now satisfies
`OBS-TH-036` with real observed data and no simulated values.

A related trap: the restart count lives at the **root** of Podman's inspect
output (`.RestartCount`), not under `.State.RestartCount` as it does in Docker.
Copying a Docker template verbatim produces a Go template error that Zabbix
rejects as an invalid numeric value.

## Finding: `podman stats` Returns Formatted Strings

`podman stats --no-stream --format json` does not return numbers:

```json
"cpu_percent": "48.12%",  "mem_percent": "99.96%",  "mem_usage": "268.3MB / 268.4MB"
```

Zabbix needs numerics, so each of those items carries a regular-expression
preprocessing step that strips the suffix. This is handled natively in the
template rather than by parsing inside the script — preprocessing runs on the
Zabbix server, so the monitored host does no extra work.

`podman stats` also emits terminal control sequences before its JSON, which is
invisible in a terminal but would corrupt the payload piped into the agent. The
script slices the output from the first `[` to guard against it.

One useful consequence: `mem_percent` is measured **against the container's
configured limit**, which is exactly what `OBS-TH-037` asks for. CPU is
reported as a percentage of one core rather than of the container's quota, so
the CPU threshold is expressed through template macros and documented as an
approximation rather than silently presented as percent-of-limit.

## Environment Variation: One Template, Every Host

The architect's PDF requires templates that adapt to what is actually present
on a host rather than assuming every host runs containers.

This is solved in the first three lines of the script. On a host with no
Podman it returns a valid, empty document:

```json
{"present":0,"runtime_ok":0,"containers":[],"stats":[],"df":[]}
```

Discovery then finds nothing, no item prototype is instantiated, no trigger
fires, and no error appears. The same template can be linked to every RHEL host
in the estate — with containers, without, or with containers added later — and
it adapts on its own. The runtime trigger is gated on `present=1` so a host
without Podman never alarms about a runtime it was never supposed to have.

## Architectural Concept: Why 7 Requirements Produce 44 Items

In the GUI the template looks far larger than its requirement count. That is
Low-Level Discovery working as intended:

```
 8 host items  + (9 prototypes x N containers)
 1 host trigger + (8 prototypes x N containers)
```

With four containers that is 44 items and 33 triggers. Add a fifth container
and nine more items appear automatically, with no template edit — which is the
whole point of discovery.

Requirements and items are also not one-to-one. `OBS-F-088` alone needs three
items (`exited`, `exitcode`, `startedat`) because its trigger logic spans all
three, in the same way the Host Access cluster stacks `SEC-F-003` triggers onto
`SEC-F-002` items.

To keep this traceable, **every item carries a `requirement` tag** with its
`OBS-F-xxx` identifier. Filtering *Latest data* by that tag shows exactly which
items implement a given requirement — the audit trail for anyone asking where a
requirement was satisfied.
