# General Information: Zabbix Architecture Concepts

This document serves as a repository for core Zabbix architectural questions and concepts.

---

## 1. How UserParameters Actually Work
**Question:** *How does the UserParameter work? Does the command get executed by the agent constantly? How does the template collect it?*

**Answer:** The Zabbix Agent is completely passive when it comes to scheduling. It does **not** execute your `UserParameter` commands on a loop by itself. The schedule is entirely dictated by the **Zabbix Template**.

Here is the exact lifecycle of how it works:
1. **The Schedule:** In the Zabbix Web GUI, your Template defines the "Update interval" (e.g., `1m` or `10m`).
2. **The Task List:** Because your agent is configured for **Active Checks**, the agent reaches out to the server and asks, *"What is my task list?"*
3. **The Execution:** The Server tells the Agent: *"Run the key `kvm.vm.cpu.time[alma10]` every 1 minute."*
4. **The Bash Command:** Every 1 minute, the Zabbix Agent looks at its local `virtualisation.conf` file, finds the `UserParameter` that matches that key, spawns a hidden Linux shell, executes your `virsh` bash command, and securely pushes the raw output text back to the Server.

---

## 2. How Native Checks Work
**Question:** *How do Native checks work? Does the agent have native UserParameters hidden inside it?*

**Answer:** Native Checks do **not** use UserParameters or Bash scripts at all. 

The Zabbix Agent 2 is a highly advanced, compiled Golang application. Inside its core source code, Zabbix developers have hardcoded hundreds of "Native Plugins" (e.g., `systemd.unit.info`, `vfs.fs.size`, `vm.memory.size`).

Here is how they differ from UserParameters:
1. The Zabbix Template tells the Agent to run `systemd.unit.info[libvirtd]`.
2. The Agent looks at its configuration file and sees no UserParameter for it. 
3. Instead of failing, the Agent checks its internal Golang code, recognizes the `systemd` plugin, and directly asks the Linux Kernel or Systemd API for the status.

**Why Native Checks are Superior:** 
Because Native Checks communicate directly with the Linux Kernel via Go/C code, they do not need to spawn a `/bin/bash` terminal. They do not need to run `grep`, `awk`, or `cut`. This makes Native Checks exponentially faster and consumes almost zero CPU on your hypervisor. 

As a Senior Engineer, you always use Native Checks first. You only architect a custom `UserParameter` (like we did for KVM and LVM) when Zabbix does not have a native plugin built-in for that specific software!

---

## 3. The Anatomy of a UserParameter
**Question:** *In the configuration file, what is `kvm.vm.state`? Is it a variable? A command?*

**Answer:** You are exactly right—it is **NOT** a command! It is what Zabbix calls an **Item Key**. 

Think of it as a custom label, a variable name, or a "trigger word" that we completely invented.

Every UserParameter is split into two halves, separated by a comma:
`UserParameter=<Item_Key>, <Linux_Bash_Command>`

**Example:**
`UserParameter=kvm.vm.state[*], sudo /usr/bin/virsh domstate $1`

1. **The Left Side (`kvm.vm.state[*]`):** This is just a label. The Zabbix Server uses this label to ask the Agent for data over the network.
2. **The Right Side (`sudo /usr/bin/virsh...`):** This is the actual Linux command.

When the Zabbix Server asks the Agent for `kvm.vm.state[alma10]`, the Agent searches its file for that exact label, grabs the Linux command attached to the right side of the comma, injects `alma10` into `$1`, and runs the code!

---

## 4. Virtual Networking: virbr0 vs vnet0
**Question:** *What is virbr0 and vnet0? How are they different and how do they act in a hypervisor?*

**Answer:** When you virtualize a server, you also have to virtualize the networking hardware. In KVM/libvirt, this is handled by two distinct components: the Virtual Bridge (`virbr`) and the Virtual Network Interface (`vnet`).

**1. The Virtual Bridge (`virbr0`) = The Network Switch**
Think of `virbr0` as an invisible, physical **Network Switch** sitting inside your hypervisor. 
- When you install KVM, it automatically creates `virbr0` (usually called the "default network").
- This virtual switch has a built-in router, handles DHCP (giving IP addresses to VMs), and performs NAT so your VMs can access the outside internet. 
- It runs 24/7 as long as the hypervisor is on, regardless of whether any VMs are running.

**2. The Virtual Interface (`vnet0`) = The Ethernet Cable**
Think of `vnet0` as the **Ethernet Cable** connecting a specific Virtual Machine to the Virtual Switch.
- Every time you power on a Virtual Machine (like `alma10`), KVM creates a temporary virtual network cable called `vnet0` (or `vnet1`, `vnet2`, etc.).
- One end of `vnet0` plugs into the VM, and the other end plugs into `virbr0`.
- **The catch:** These are highly volatile. When you shut off the `alma10` VM, the hypervisor unplugs and destroys the `vnet0` cable entirely. It will magically reappear the next time the VM boots.

**Why Zabbix Monitoring Matters Here:**
If `virbr0` goes down, **ALL** of your VMs lose internet access (the switch crashed). 
If `vnet0` is dropping packets, only **ONE** specific VM is having connectivity issues (a bad cable). By filtering our Zabbix LLD to monitor both `virbr` and `vnet`, we can instantly pinpoint if a network outage is a global hypervisor failure or just an isolated VM issue!

**Follow-up Question:** *Why would `virbr1` or `virbr2` exist? What causes them to be created?*

**Answer:** 
`virbr0` is simply the "default" network that KVM creates out-of-the-box. If you see a `virbr1` or `virbr2`, it means a System Administrator explicitly created a **second, entirely separate Virtual Switch** for architectural reasons.

Here is why an engineer would create `virbr1`:
1. **Network Isolation (DMZ):** You might want your Web Server VMs to be on `virbr0` (with public internet access), but your sensitive Database VMs on `virbr1` (an isolated, host-only network with no internet). Since they are plugged into different "switches," they cannot talk to each other.
2. **VLAN Mapping:** A senior network engineer might bind `virbr0` to VLAN 10 (Management), and `virbr1` to VLAN 20 (Production).

*(Note: The Zabbix Regex filter we engineered is `virbr[0-9]+`. The `[0-9]+` tells Zabbix to match ANY number. If a SysAdmin creates `virbr1`, `virbr2`, or `virbr99`, Zabbix will automatically discover and monitor them without us ever having to touch the code again!)*

---

## 5. Why do LLD Items only appear after data is pulled?
**Question:** *Why is it so hard to find newly added items? When I look at the host, they aren't there. They only appear in Latest Data after data is pulled. Why?*

**Answer:** This is the most confusing (but most powerful) part of Zabbix. It happens because we are using **Low-Level Discovery (LLD)**.

There are two types of items in Zabbix:
1. **Static Items:** (Like `system.cpu.util`). These are hardcoded. As soon as you link a template, they permanently appear in your Host's item list, even if the agent is turned off.
2. **Item Prototypes (Blueprints):** This is what we built for KVM. `kvm.vm.disk.capacity[{#VMNAME}]` is **not** a real item. It is just a blueprint. 

**The LLD Lifecycle:**
- When you import the template, Zabbix saves the blueprints, but it does *not* create any actual items on the host yet.
- The Zabbix Server asks the Agent: *"Run `kvm.vm.discovery`."*
- The Agent replies: *"I found a VM named `alma10`!"*
- The Zabbix Server takes the blueprint, stamps the word `alma10` into it, and **dynamically creates a brand new, real item:** `kvm.vm.disk.capacity[alma10]`.
- Because it was just created dynamically, it instantly shows up in Latest Data!

**Where to find them in the GUI:**
If you want to see the configuration of these items *before* they are discovered, you cannot look in the normal "Items" menu. You must go to:
**Data Collection -> Hosts -> [Your Host] -> Discovery Rules -> Click "Item Prototypes"**
