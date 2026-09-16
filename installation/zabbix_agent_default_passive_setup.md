# Default Zabbix Agent Installation (Passive Mode)

## Overview
This document outlines the standard, out-of-the-box Zabbix Agent deployment. In this "Passive" model, the central Zabbix Server acts as the initiator. It connects to the agent's listening port (10050) on a strict schedule, asks for specific metrics, and waits for the plain-text reply.

---

## Phase 1: Installation
To simulate a fresh default installation from scratch, you must install the repository and the agent package.

**1. Add Zabbix 8.0 Repository**
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/8.0/release/rhel/9/noarch/zabbix-release-latest-8.0.el9.noarch.rpm
```

**2. Clean Package Cache**
```bash
dnf clean all
```

**3. Install Zabbix Agent 2**
```bash
dnf install zabbix-agent2 zabbix-selinux-policy -y
```

---

## Phase 2: Firewall Configuration
Because the Zabbix Server initiates the connection, the target VM must have its firewall opened to allow inbound traffic from the server.

**1. Open the Agent Port**
Permanently open TCP port 10050 to allow the Zabbix Server to connect.
```bash
firewall-cmd --add-port=10050/tcp --permanent
```

**2. Reload Firewall**
Apply the new rule immediately.
```bash
firewall-cmd --reload
```

---

## Phase 2: Agent Configuration

**1. Open Configuration File**
```bash
vi /etc/zabbix/zabbix_agent2.conf
```

**2. Apply Default Settings**
In a passive setup, we tell the agent to listen for instructions from the Zabbix Server's IP address. No encryption parameters are used.
```ini
# The IP address of your Zabbix Server. The agent will ONLY accept inbound requests from this IP.
Server=192.168.20.10

# The identity of this specific host
Hostname=Rhel9-Not-Hardened

# (Ensure ServerActive and all TLS parameters are commented out with a #)
```

**3. Restart the Agent**
Apply the configuration changes.
```bash
systemctl restart zabbix-agent2
```

---

## Phase 3: Zabbix Web UI Registration

**1. Create/Edit the Host**
In the Zabbix Web UI, navigate to **Data collection -> Hosts**.

**2. Configure the Host Tab**
* **Host name:** `Rhel9-Not-Hardened`
* **Templates:** Select the standard **`Linux by Zabbix agent`** template (Do *not* select the 'active' variant).
* **Interfaces:** Add an Agent interface with the VM's IP address (e.g., `192.168.20.20`). In passive mode, the Zabbix Server heavily relies on this IP address to know where to send its requests.

**3. Configure the Encryption Tab**
* **Connections to host:** Check `No encryption`
* **Connections from host:** Check `No encryption`

**4. Finalize**
Click **Update** or **Add**. The green "ZBX" availability icon in the dashboard should turn green almost immediately as the server successfully performs its first inbound poll on port 10050.

---

## Phase 4: Advanced - Dual Mode (Passive + Active)
While this document outlines a strictly Passive setup, a single Zabbix Agent is capable of running **both Passive and Active engines simultaneously**. This is incredibly useful if you have attached multiple templates to a single host that use mixed protocols (e.g., a standard Passive OS template alongside a custom Active application template).

**How to Enable Dual-Mode**
To allow the agent to answer server pull requests *and* actively push its own data, simply enable both server directives in `/etc/zabbix/zabbix_agent2.conf`:

```ini
# Enables the Passive engine (Allows the server to pull data from the agent)
Server=192.168.20.10

# Enables the Active engine (Tells the agent where to push data)
ServerActive=192.168.20.10
```

After modifying the file, run `systemctl restart zabbix-agent2`. The agent will immediately begin accepting passive inbound connections on port 10050 while simultaneously managing its own outbound active checks on port 10051.
