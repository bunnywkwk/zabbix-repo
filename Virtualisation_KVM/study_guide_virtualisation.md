# Zabbix Engineering Study Guide: Virtualisation (KVM)

This document is a personal reference guide. It breaks down the exact `UserParameter` scripts used in the KVM PoC, explains the Linux Bash pipeline logic, and details how the Zabbix Server Template processes the data.


## CIS-Hardened File Permissions (The umask 027 Trap)

**The Architectural Problem:**
In a standard Linux environment, the default `umask` is `022`. When `root` creates a configuration file, it defaults to `644` permissions, meaning the `zabbix` background service can read it natively. However, in a CIS-Hardened RHEL environment, the OS enforces a strict `umask 027`. This means files created by `root` default to `640` and are owned by `root:root`. This completely locks out the `zabbix` user, causing a "Permission Denied" service crash.

**The Security Architect Solution:**
To allow Zabbix to read the file while maintaining strict security boundaries to prevent Privilege Escalation, we used a secure group-ownership model:
```bash
sudo chown root:zabbix /etc/zabbix/zabbix_agent2.d/virtualisation.conf
sudo chmod 640 /etc/zabbix/zabbix_agent2.d/virtualisation.conf
```
- **`root` (Owner):** Has Read/Write access. (This prevents a compromised Zabbix agent from maliciously rewriting its own configuration).
- **`zabbix` (Group):** Has Read-Only access.
- **Others:** Have ZERO access.

---


## Virtual Machine Discovery
**Agent Code (`virtualisation.conf`):**
```ini
UserParameter=kvm.vm.discovery, sudo /usr/bin/virsh list --all --name | grep -v '^$' | sed -e 's/^/{"{#VMNAME}":"/' -e 's/$/"}/' | paste -sd, | sed -e 's/^/[/' -e 's/$/]/'
```

**How the Linux Code Works:**
- `virsh list --all --name`: Asks the hypervisor for a raw list of VM names (e.g., `alma10`).
- `grep -v '^$'`: Strips out any invisible blank lines.
- `sed -e ...`: Wraps the VM name in Zabbix JSON syntax `{"{#VMNAME}":"alma10"}`.
- `paste -sd,`: Joins multiple VMs together with a comma.
- `sed -e ...`: Wraps the entire string in `[` and `]` to create a valid JSON array.

**How the Zabbix Template Uses It:**
- **Component:** Low-Level Discovery (LLD) Rule.
- **Action:** The Zabbix Server pulls this JSON array. For every `{#VMNAME}` it finds, it dynamically spawns new Items and Triggers specifically for that VM.

---

## VM Power State Monitoring
**Agent Code:**
```ini
UserParameter=kvm.vm.state[*], sudo /usr/bin/virsh domstate $1
```

**How the Linux Code Works:**
- The `[*]` tells the agent to accept variables from the Zabbix Server.
- If Zabbix asks for `kvm.vm.state[alma10]`, the agent replaces `$1` with `alma10` and runs: `sudo /usr/bin/virsh domstate alma10`.
- The hypervisor returns a simple text string like `running` or `shut off`.

**How the Zabbix Template Uses It:**
- **Component:** Item Prototype & Trigger Prototype.
- **Action:** Zabbix collects the text. The Trigger uses the `find()` function to check the text. If `find(..., "shut off")=1`, it fires a WARNING alert.

---

## Guest CPU Utilisation
**Agent Code:**
```ini
UserParameter=kvm.vm.cpu.time[*], sudo /usr/bin/virsh domstats $1 --cpu-total | grep "cpu.time=" | cut -d= -f2
```

**How the Linux Code Works:**
- `virsh domstats alma10 --cpu-total`: Outputs hypervisor statistics.
- `grep "cpu.time="`: Isolates the specific CPU time line (e.g., `cpu.time=123456789`).
- `cut -d= -f2`: Splits the string at the `=` sign and grabs the 2nd piece, returning just the raw number. Note: This number is a continuously increasing counter in *nanoseconds*.

**How the Zabbix Template Uses It:**
- **Component:** Item Prototype with **Preprocessing**.
- **Action:** Because a raw nanosecond counter is useless on a graph, Zabbix Preprocessing fixes it:
  1. `Multiplier (1e-09)`: Converts nanoseconds to seconds.
  2. `Change per second`: Subtracts the previous poll from the current poll. This perfectly translates a running counter into "CPU Seconds consumed per second" (Live CPU Utilisation).

---

## Guest Memory (RAM) Utilisation
**Agent Code:**
```ini
UserParameter=kvm.vm.ram.used[*], sudo /usr/bin/virsh domstats $1 --balloon | grep "balloon.current=" | cut -d= -f2
```

**How the Linux Code Works:**
- Operates identically to the CPU script, but isolates `balloon.current=`.
- The hypervisor outputs this metric in **Kibibytes (KiB)**.

**How the Zabbix Template Uses It:**
- **Component:** Item Prototype with **Preprocessing**.
- **Action:** Zabbix uses a Preprocessing `Multiplier (1024)` to convert KiB into standard Bytes. The Item is configured with `units: B`, so the Zabbix GUI automatically formats it beautifully (e.g., `2.5 GB`).

---

## Storage Pool Discovery & Capacity
**Agent Code:**
```ini
UserParameter=kvm.pool.discovery, sudo /usr/bin/virsh pool-list --all --name | ... (JSON Wrapper) ...
UserParameter=kvm.pool.capacity[*], sudo /usr/bin/virsh pool-info $1 --bytes | grep "Capacity:" | tr -s ' ' | cut -d' ' -f2
UserParameter=kvm.pool.available[*], sudo /usr/bin/virsh pool-info $1 --bytes | grep "Available:" | tr -s ' ' | cut -d' ' -f2
```

**How the Linux Code Works:**
- `pool-info $1 --bytes`: Forces `virsh` to output exact bytes instead of human-readable GiB (which is extremely hard to parse in bash).
- `tr -s ' '`: "Squeeze Spaces" - compresses multiple spaces into a single space.
- `cut -d' ' -f2`: Splits the string by that single space to extract the raw byte number.

**How the Zabbix Template Uses It:**
- **Component:** LLD Rule & Trigger Prototype.
- **Action:** Zabbix discovers pools like `default`. It pulls Capacity and Available bytes. The Trigger uses math: `last(available) / last(capacity) * 100 < 10` to fire a CRITICAL alert if the virtual hard drive pool falls below 10% free space.

---

## Cross-OS Systemd Architecture (No Agent Code)
**How it Works:**
Instead of writing a custom `UserParameter` to check if `libvirtd` (RHEL 9) or `virtqemud` (RHEL 10) is running, we used Native Zabbix LLD.

**How the Zabbix Template Uses It:**
- **Component:** `systemd.unit.discovery[service]`
- **Action:** The Zabbix Server asks the OS for every running systemd service. 
- **Filter:** The template applies a strict Regex filter: `^(libvirtd\.service|virtqemud\.service)$`. 
- **Result:** It dynamically adapts to the host OS architecture without requiring a single line of custom bash script on the Zabbix Agent!

## Virtual Network Component State (virbr0 & vnet0)
**How it Works (No UserParameters Needed!):**
KVM virtual bridges (`virbr0`) and guest network interfaces (`vnet0`) are exposed directly to the Linux kernel network stack. We bypassed custom bash scripts completely and used Zabbix's native `net.if.discovery` plugin.

**How the Zabbix Template Uses It:**
- **Component:** LLD Filter & Native Item Prototypes
- **Action:** The LLD filter `^(virbr[0-9]+|vnet[0-9]+)$` forces Zabbix to ignore standard physical NICs (like `eth0` or `ens192`) and exclusively discover the virtual networks. It then automatically tracks their traffic using native `net.if.in` and `net.if.out` keys.

---

## Guest Lifecycle & Error Log Monitoring
**Zabbix Key Used:** 
- `log[/var/log/libvirt/qemu/{#VMNAME}.log,"(?i)(error|failed|shutting down|starting)"]`

**The Architectural Secret (Item Multiplexing):**
Instead of creating one item to track Guest Lifecycles and a second item to track Boot Crashes, we combined them into a single Data Item. This prevents the Zabbix Agent from opening the same `alma10.log` file twice every second (which wastes Disk I/O). Zabbix pulls the stream once, and we use multiple distinct Triggers to parse the text in the Zabbix Database.

---

## Deep Virtual Storage & Thin Provisioning Ratios
**Agent Code:**
```ini
UserParameter=kvm.vm.disk.capacity[*], sudo /usr/bin/virsh domstats $1 --block | grep "block.0.capacity=" | cut -d= -f2
UserParameter=kvm.vm.disk.allocation[*], sudo /usr/bin/virsh domstats $1 --block | grep "block.0.allocation=" | cut -d= -f2
```

**How the Linux Code Works:**
- `virsh domstats $1 --block`: Outputs deep block-device metrics for a specific VM.
- We use `grep` and `cut` to isolate the `block.0` disk, grabbing its Provisioned maximum size (Capacity) and its actual physical size on the hard drive (Allocation).

**How the Zabbix Template Uses It:**
- **Action:** To satisfy the strict Annex A requirement, we don't just graph the numbers. We use a Zabbix trigger expression: `last(capacity) / last(allocation) > 1.5` to automatically trigger an alert if the Thin Provisioning ratio spirals out of control.

---

## Active Snapshot Monitoring
**Agent Code:**
```ini
UserParameter=kvm.vm.snapshot.count[*], sudo /usr/bin/virsh snapshot-list $1 --name | grep -v '^$' | wc -l
```

**How the Linux Code Works:**
- Extracts a list of snapshot names. `grep -v '^$'` removes blank lines, and `wc -l` simply counts the number of lines. If a VM has no snapshots, it returns `0`.

**How the Zabbix Template Uses It (The Aging Trick):**
Trying to calculate snapshot age in a Bash script is a nightmare of date formatting. Instead, we use Zabbix's Native History functions! 
- **Trigger:** `min(/Scope Virtualisation KVM/kvm.vm.snapshot.count[{#VMNAME}],72h)>0`
- **Translation:** If the Snapshot count has been greater than 0 continuously for the last 72 hours, fire a Warning! This tracks snapshot age perfectly with zero CPU overhead on the hypervisor.
