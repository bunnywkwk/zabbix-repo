# Implementation Matrix: Virtualisation (KVM)

This matrix evaluates the 18 requirements under the `Virtualisation (KVM)` cluster. 

### Engineering Philosophy for this Cluster
Unlike VMware, Zabbix Agent 2 does not possess native Go plugins for KVM (`libvirt`). To maintain our strict security posture on CIS-hardened RHEL 9/10 hypervisors, we will split this architecture:
1. **Native OS Checks:** We will use native Zabbix keys to monitor the Hypervisor services, host capacity, and virtual bridge interfaces.
2. **Restricted UserParameters:** For guest-specific metrics, we will engineer a Low-Level Discovery (LLD) script that uses strict `sudoers` rules to execute read-only `virsh` commands, completely avoiding the dangerous `system.run` feature.

| ID | Title | Approach | Achievable? & Justification |
| :--- | :--- | :--- | :--- |
| **OBS-F-035** | Virtualisation Management Service | Native Check (`systemd.unit.info`) | **Yes.** Zabbix natively queries systemd to ensure `libvirtd.service` (or `qemu-kvm`) is active and running. |
| **OBS-F-036** | Hypervisor Capacity & Allocation | Native Checks (`system.cpu`, `vm.memory`) | **Yes.** The hypervisor's total available resources are monitored using standard native OS keys. |
| **OBS-F-037** | Guest Inventory Monitoring | `UserParameter` + LLD (`virsh list`) | **Yes (Requires UserParameter).** A custom LLD script will query `virsh list --all` to dynamically discover all VMs hosted on the hypervisor. |
| **OBS-F-038** | Guest Power State Monitoring | `UserParameter` (`virsh domstate`) | **Yes (Requires UserParameter).** Mapped to the LLD discovery to track if each VM is `running`, `paused`, or `shut off`. |
| **OBS-F-039** | Guest Lifecycle Event Recording | Native Log Check (`log[/var/log/libvirt]`) | **Yes.** We can natively monitor libvirt logs for lifecycle events, provided we set a secure ACL (just like Cluster 1). |
| **OBS-F-040** | Guest Configuration Change | Native Audit Log / File Check | **Partial.** True config change monitoring requires `auditd` rules monitoring `/etc/libvirt/qemu/`. Zabbix can monitor the audit log natively. |
| **OBS-F-041** | Guest Start Failure Detection | Native Log Check (`log[/var/log/libvirt]`) | **Yes.** Failed starts instantly print critical errors to the qemu logs, which Zabbix parses natively. |
| **OBS-F-043** | Guest Responsiveness | `UserParameter` (`virsh qemu-agent-command`) | **Yes (Requires UserParameter).** Requires the QEMU Guest Agent to be installed on the VMs. |
| **OBS-F-044** | Guest Resource Utilisation | `UserParameter` (`virsh domstats`) | **Yes (Requires UserParameter).** Queries hypervisor-observed CPU/Mem/Block stats for each VM. |
| **OBS-F-045** | Guest Resource Contention | `UserParameter` (`virsh domstats`) | **Yes (Requires UserParameter).** Tracks ballooning and CPU steal time. |
| **OBS-F-046** | Storage Pool Monitoring | `UserParameter` (`virsh pool-info`) | **Yes (Requires UserParameter).** Discovers and tracks KVM storage pool capacity. |
| **OBS-F-047** | Virtual Disk Image Monitoring | `UserParameter` (`virsh vol-list`) | **Yes (Requires UserParameter).** |
| **OBS-F-048** | Thin Provisioned Allocation | `UserParameter` (`virsh vol-info`) | **Yes (Requires UserParameter).** Tracks actual bytes consumed versus logical provisioned size. |
| **OBS-F-049** | Snapshot Monitoring | `UserParameter` (`virsh snapshot-list`) | **Yes (Requires UserParameter).** Triggers alerts if snapshots are left orphaned or grow too large. |
| **OBS-F-053** | Orphaned Disk Image Detection | External Script / Audit | **Partial.** Difficult to do efficiently in real-time via the Zabbix Agent. Best achieved via a daily cron job that pushes to Zabbix Trapper. |
| **OBS-F-050** | Virtual Network Component State | Native Check (`net.if.discovery`) | **Yes.** Zabbix natively discovers KVM bridges (e.g., `virbr0`) and vNICs (e.g., `vnet0`) as standard network interfaces! |
| **OBS-F-051** | Virtual Network Throughput/Errors | Native Check (`net.if.in/out`) | **Yes.** Because Zabbix sees virtual bridges as standard interfaces, we can natively monitor their traffic and dropped packets. |
| **OBS-F-052** | Virtual Network Conformance | Custom Script | **Partial.** Checking if a VM is attached to the *wrong* network segment requires complex custom scripting and baseline comparison. |
