# Zabbix Template Mapping Plan

Based on the `Working instructions — Observability requirements → Zabbix.pdf`, your mentor has defined **6 distinct Requirement Clusters**. The goal is to build exactly one master Zabbix template per cluster. 

This document maps out the strategy for our first template: **Host Health & Availability**. It pulls the exact requirements and numeric thresholds directly from the `Annex A` spreadsheet.

---

## Template 1: `RHEL Host Health & Availability`

### 1. Resource Utilisation (OBS-F-003)
* **Goal:** Monitor CPU, memory, swap, and I/O.
* **Extraction Strategy:** Native Zabbix Agent Keys (`system.cpu.util`, `vm.memory.size`, `system.swap.size`, `system.cpu.util[,iowait]`). 100% secure, zero shell execution.
* **Defined Thresholds (from Annex A):**
  * **Processor utilisation:** Warning `> 80%` (5 min) | Critical `> 95%` (5 min)
  * **Available memory:** Warning `< 20%` | Critical `< 10%`
  * **Swap activity rate:** Warning on *any sustained activity* for 5 min
  * **Processor I/O wait:** Warning `> 20%` (5 min) | Critical `> 40%` (5 min)

### 2. Service Status Monitoring (OBS-F-004)
* **Goal:** Detect service start, stop, restart, and failure.
* **Extraction Strategy:** Native Key (`systemd.unit.info`).
* **Defined Thresholds:** No percentage, alert triggers immediately if state `!= active`.

### 3. File Descriptor Utilisation (OBS-F-018)
* **Goal:** Monitor open file descriptor counts against limits.
* **Extraction Strategy:** Native Key (`kernel.maxfiles` and `vfs.file.time`).
* **Defined Thresholds (from Annex A):**
  * Warning: `> 70%` of configured limit
  * Critical: `> 85%` of configured limit

### 4. Critical Process Monitoring (OBS-F-016)
* **Goal:** Monitor specific critical processes for absence or runaway resource consumption.
* **Extraction Strategy:** Native Key (`proc.num` and `proc.mem`).
* **Defined Thresholds (from Annex A):**
  * **CPU:** Warning `> 80%` of one core for 10 min
  * **Memory:** Warning `> 80%` of host memory

### 5. Service Restart Loop Detection (OBS-F-017)
* **Goal:** Detect if a service is crash-looping.
* **Extraction Strategy:** Native Key (analyzing `systemd` uptime/restart counts).
* **Defined Thresholds (from Annex A):**
  * Warning: `> 3` restarts in 10 minutes
  * Critical: `> 5` restarts in 10 minutes

### 6. Linux Host State & Unreachable Detection (OBS-F-001)
* **Goal:** Detect unreachable/stopped states.
* **Extraction Strategy:** Zabbix internal `agent.ping` and `nodata()` triggers.
* **Defined Thresholds:** Alert if agent stops communicating for `> 3` minutes.

---

## Next Steps for Automation
1. We will write a single `.yaml` file containing all of the items and triggers mapped above for Template 1.
2. We will use **Ansible** to automatically push this template into your Zabbix server, satisfying the "Automated import" instruction from your PDF. 
3. We will repeat this grouping process for Template 2 (`Storage & Filesystem`).
