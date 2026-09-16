# Secure Metric Collection Methods & Requirement Mapping

## Overview
When engineering a Zabbix monitoring solution for hardened RHEL environments, metric extraction must strictly avoid arbitrary remote code execution (e.g., `system.run`). This document outlines the approved, secure methods for extracting data, ranked by their security posture and performance overhead.

---

## Part 1: Approved Secure Collection Methods

### 1. Native Agent Keys (Highest Security & Performance)
* **Mechanism:** Data is gathered using internal C/Go system calls natively compiled into the Zabbix Agent (e.g., `system.cpu.util`, `vfs.fs.size`).
* **Security Profile:** 100% secure. It interacts directly with the Linux kernel (via `/proc`, `/sys`, or APIs). It never spawns a bash shell, making command injection impossible.
* **Best For:** Standard OS metrics (CPU, RAM, Disk, Network, Systemd services).

### 2. Zabbix Agent 2 Custom Plugins (Go)
* **Mechanism:** Zabbix Agent 2 allows administrators to write custom plugins in the Go programming language that load directly into the agent. 
* **Security Profile:** Highly secure. Like native keys, these plugins are compiled code that execute API or system calls directly without invoking a Linux shell.
* **Best For:** Complex application monitoring (Databases, Web Servers, custom APIs) where performance and strict security are paramount.

### 3. Zabbix Trapper / Zabbix Sender
* **Mechanism:** Instead of the agent gathering data, local scheduled tasks (like a cron job) or internal applications use the `zabbix_sender` utility to push data directly to the Zabbix Server.
* **Security Profile:** Highly secure. The logic is entirely controlled by the local application or cron job on the RHEL VM. The Zabbix Server just passively accepts the incoming data string.
* **Best For:** Application logs, backup job outcomes, or long-running scripts that cannot wait for a Zabbix poll timer.

### 4. UserParameters (Secure but spawns a shell)
* **Mechanism:** The administrator hardcodes a specific bash command into `/etc/zabbix/zabbix_agent2.d/` and assigns it an alias (e.g., `UserParameter=chrony.offset,chronyc tracking...`).
* **Security Profile:** Secure. The Zabbix Server cannot alter the command; it can only request the alias. However, because it relies on spawning a local bash shell to run the script, it has slightly higher system overhead than Native Keys.
* **Best For:** Edge cases, custom scripts, or third-party tools (like Chrony) where Native Keys or Go Plugins do not currently exist.

### 5. Loadable C Modules
* **Mechanism:** Custom C code compiled as a shared object (`.so`) library and loaded into the agent at startup.
* **Security Profile:** Extremely secure and fast, but requires high development overhead to write and maintain C code.
* **Best For:** High-frequency custom metric collection in enterprise environments.

---

## Part 2: Prohibited Methods

### `system.run[*]` (Critical Security Risk)
* **Mechanism:** The Zabbix Server passes raw bash strings over the network to be executed by the agent.
* **Security Profile:** Zero security. If the Zabbix Server is compromised, threat actors gain instant Remote Code Execution (RCE) across the entire fleet of monitored servers.

---

## Part 3: Operational Requirement Mapping

*This table will be used to map the specific requirements from the Operational & Security Spreadsheet to the exact collection method utilized, providing architectural justification for the choice.*

| Requirement ID | Requirement Title | Chosen Collection Method | Justification (Why this method?) |
| :--- | :--- | :--- | :--- |
| **Req-08** | Resource Utilisation (CPU/RAM) | Native Agent Keys | Built-in native keys (`system.cpu.util`) provide the exact data with zero shell execution and lowest overhead. |
| **Req-11** | Service Status Monitoring | Native Agent Keys | `systemd.unit.info` natively queries D-Bus without requiring root shell scripts. |
| **Req-38** | Clock Accuracy (Chrony) | UserParameter | No native key exists for Chrony offsets. A hardcoded UserParameter ensures the script is locked locally on the VM. |
| *Pending* | *Pending* | *Pending* | *Pending* |
