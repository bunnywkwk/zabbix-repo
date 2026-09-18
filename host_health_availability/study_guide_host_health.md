# Zabbix Engineering Study Guide: Host Health & Availability (Cluster 1)

This document is a personal reference guide detailing how Native Checks and OS-level security modifications were engineered to monitor base Linux health without exposing the system to vulnerabilities.

---

## 1. Native CPU and Memory Polling
**Zabbix Keys Used:** 
- `system.cpu.util`
- `vm.memory.size[pavailable]`

**How it Works:**
Unlike custom scripts, these are **Native Golang Plugins** built directly into the Zabbix Agent 2 binary. When the Zabbix Server requests `system.cpu.util`, the agent bypasses the Linux bash terminal entirely and securely pulls the data directly from the Linux Kernel (`/proc/stat` and `/proc/meminfo`). This consumes practically zero CPU overhead.

---

## 2. Service Monitoring (`sshd`)
**Zabbix Key Used:** 
- `systemd.unit.info[sshd]`

**How the Trigger Works:**
The `systemd.unit.info` native plugin returns a massive block of text containing every property of the service. 
To determine if the service crashed, the Zabbix Trigger uses the `find()` function:
`find(/Host Health/systemd.unit.info[sshd],,"like","ActiveState=active")=0`
- **Translation:** "Search the text block for the string `ActiveState=active`. If you find it 0 times, fire a CRITICAL alert!"

---

## 3. Secure Event Log Monitoring (The ACL Fix)
**Zabbix Key Used:** 
- `vfs.file.regmatch[/var/log/messages, "error|critical"]`

**The Security Problem:**
By default, standard Linux users (like the `zabbix` service account) are strictly forbidden from reading `/var/log/messages`. If the Zabbix Server requested this key, the agent would return `Permission Denied` and become Unsupported.

**The Architectural Solution:**
Many junior admins make the dangerous mistake of granting the Zabbix agent `root` access or putting it in the `wheel` group to solve this. This creates a massive Privilege Escalation vulnerability.
Instead, we used **Access Control Lists (ACL)**:
```bash
setfacl -m u:zabbix:r /var/log/messages
```
- **Why this is perfect:** It surgically grants the `zabbix` user Read-Only (`r`) access to that *one specific file*, maintaining strict CIS Security compliance across the rest of the OS!
