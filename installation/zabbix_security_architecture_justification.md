# Zabbix Monitoring Architecture: Security Justification

## Executive Summary
This document outlines the architectural decisions and security justifications for the deployment of the Zabbix monitoring infrastructure across our RHEL environments. Instead of utilizing the default Zabbix installation parameters, the deployment was intentionally engineered to use **Active Checks** paired with **Pre-Shared Key (PSK) Encryption**. 

This approach ensures strict adherence to modern security standards, particularly when operating alongside hardened security tools like Wazuh.

## 1. Attack Surface Reduction (Zero Inbound Ports)
* **The Default Risk:** A standard Zabbix deployment requires opening inbound firewall port `10050` on every monitored endpoint so the central server can periodically connect and ask for data. This expands the network attack surface across the entire infrastructure.
* **Our Implementation:** By configuring Zabbix Agents to strictly use "Active Checks," the agents initiate all connections outward to the central server on port `10051`. This allows the inbound firewall rules on all target VMs to remain completely closed, drastically reducing visibility to network scanners and internal threat actors.

## 2. Data Confidentiality in Transit
* **The Default Risk:** Out-of-the-box Zabbix configurations transmit system metrics in plain text. This can expose sensitive internal data—such as active process names, routing tables, and filesystem structures—to anyone sniffing the network.
* **Our Implementation:** All communication between the Zabbix Agents and the Zabbix Server is mandated to use TLS encryption backed by a 64-character cryptographic Pre-Shared Key (PSK). This ensures all metric data is fully scrambled in transit, guaranteeing data privacy across the network.

## 3. Scalability and Polling Overhead
* **The Default Risk:** In a passive polling model, the central Zabbix Server assumes the computational overhead of managing schedules, initiating network connections, and waiting for replies from every single endpoint. This creates a severe performance bottleneck as the environment grows.
* **Our Implementation:** Active Checks offload the scheduling and data-gathering workload entirely to the endpoint agents. The central server simply receives and processes the incoming metrics, allowing the architecture to seamlessly scale to thousands of endpoints without degrading central server performance.

## 4. Cloud and NAT Agility
* **The Default Risk:** Passive central polling fails when endpoints are located behind strict Network Address Translation (NAT) boundaries, in isolated cloud Virtual Private Clouds (VPCs), or on networks lacking public routing.
* **Our Implementation:** Because Active Checks rely entirely on outbound egress traffic (which is traditionally permitted even in strict environments), the agents can successfully traverse NAT boundaries and strict cloud firewalls. This allows metrics to be delivered reliably without requiring complex reverse-proxy or VPN engineering.

## 5. Scheduling and Polling Mechanics (Active vs Passive)
* **The Default Risk (Passive Scheduling):** In a standard configuration, the Zabbix Server acts as a micromanager. It reads the specific "Update interval" timers defined in the templates (e.g., checking CPU every 1 minute) and physically initiates a new network connection to the agent every time a timer hits zero. This generates excessive network chatter and server CPU strain.
* **Our Implementation (Autonomous Active Agents):** In our setup, the Agent connects to the Server periodically (default every 2 minutes via `RefreshActiveChecks`) and downloads the entire template schedule. The Agent then disconnects, holds the "stopwatches" locally, and autonomously gathers the data at the exact times requested by the template. It then securely pushes the buffered answers to the server. This guarantees that the template's strict timing rules are perfectly respected without forcing the server to manage the network polling.

## Conclusion
By rejecting the default passive and unencrypted configuration in favor of an Active/PSK architecture, the monitoring infrastructure achieves a highly secure posture. It successfully coexists with hardened security agents like Wazuh without introducing new network vulnerabilities, ensuring the environment remains both deeply observable and strictly secured.
