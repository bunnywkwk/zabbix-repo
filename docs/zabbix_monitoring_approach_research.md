# Research & Justification: Zabbix Monitoring Approaches

This document outlines the research conducted into various Zabbix data collection methods, weighs them against RHEL security best practices, and justifies the chosen approach for the initial requirement clusters.

## 1. Monitoring Approaches Analyzed

### A. Native Agent Checks
* **How it works:** Data is gathered using internal system calls (C/Go) natively compiled into the Zabbix Agent (e.g., `system.cpu.util`).
* **Security Posture (Excellent):** The agent interacts directly with the Linux kernel (via `/proc`, `/sys`, or APIs). It **never** spawns a bash shell, making command injection mathematically impossible. It requires no elevated privileges (sudo/root) to read standard OS performance metrics.

### B. UserParameters
* **How it works:** The administrator hardcodes a specific bash script locally on the RHEL machine (`/etc/zabbix/zabbix_agent2.d/`) and assigns it an alias. The Zabbix server calls the alias.
* **Security Posture (Good):** Secure, because the script is locally controlled and locked down by the Linux admin. The Zabbix server cannot alter the command. However, it does require spawning a local bash shell, which introduces slight overhead.

### C. Zabbix Agent 2 Plugins (Go)
* **How it works:** Custom plugins written in Go that compile directly into the agent memory space.
* **Security Posture (Excellent):** Highly secure and performant, similar to native checks. No shell execution. Requires high development effort.

### D. External Scripts / `system.run`
* **How it works:** The Zabbix server sends raw, arbitrary bash strings across the network for the agent to execute on the host.
* **Security Posture (Critical Risk - REJECTED):** Violates least privilege and security best practices. If the central Zabbix server is compromised, threat actors gain instant Remote Code Execution (RCE) across all monitored RHEL nodes.

---

## 2. Chosen Approach & Justification

### Cluster 1: Host Health & Availability
* **Chosen Approach:** **Native Agent Checks**
* **Justification:** For standard host health (CPU, Memory, Systemd services, Filesystems), Zabbix has robust Native Agent Checks. We are choosing this approach because it strictly adheres to the principle of least privilege. By utilizing native checks, we completely avoid introducing unmaintained third-party scripts, we do not need to grant the `zabbix` user sudo/root access, and we ensure zero shell execution on the host system.

### Cluster: Application / Custom Metrics (e.g., Chrony Time Accuracy)
* **Chosen Approach:** **UserParameters**
* **Justification:** Where Native Checks do not exist (e.g., parsing `chronyc` tracking output), we will utilize UserParameters. This ensures the execution logic remains locally hardcoded and auditable on the RHEL endpoint, preventing arbitrary code injection from the network while still avoiding unmaintained third-party plugins.
