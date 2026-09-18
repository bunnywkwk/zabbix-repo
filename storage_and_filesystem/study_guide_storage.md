# Zabbix Engineering Study Guide: Storage & Filesystem (Cluster 2)

This document is a personal reference guide detailing how Native LLD and custom `UserParameter` pipelines were engineered to dynamically map volatile physical and logical storage architectures.

---

## 1. Dynamic Filesystem Discovery (Native LLD)
**Zabbix Key Used:** 
- `vfs.fs.discovery`

**How it Works:**
Instead of hardcoding standard drives like `/` or `/data`, we used Zabbix's Native Filesystem Discovery plugin. The agent automatically scans the OS and returns a JSON array of every mounted drive. 

**The Regex Filter:**
Linux creates dozens of noisy, fake virtual filesystems (like `tmpfs`, `sysfs`, `proc`). To prevent Zabbix from monitoring garbage data, we applied a strict **LLD Filter** on the `{#FSTYPE}` macro:
`^(ext4|xfs|btrfs)$`
- **Translation:** "Only create monitoring items if the discovered drive is physically formatted as EXT4, XFS, or BTRFS."

---

## 2. Advanced Filesystem Triggers
We deployed two highly advanced Triggers based on the discovered drives:

**A. Predictive Exhaustion (Capacity Trend)**
`timeleft(/Storage/vfs.fs.size[{#FSNAME},pfree],1h,0) < 7d`
- **Translation:** Zabbix analyzes the historical disk-write speed over the last `1h`. If the math predicts the drive will hit `0%` free space in less than `7 days`, it fires a predictive warning!

**B. Read-Only Crash Detection**
`find(/Storage/vfs.file.regmatch[/proc/mounts, "{#FSNAME} ro "],,"like","1")=1`
- **Translation:** When a physical hard drive starts failing, Linux automatically remounts it as `Read-Only (ro)` to protect data. This trigger constantly scans `/proc/mounts` and fires a CRITICAL alert the exact second the drive drops into `ro` mode.

---

## 3. Logical Volume Monitoring (LVM)
Because Zabbix does not possess a native plugin for backend Logical Volumes, we engineered custom `UserParameters`.

**Agent Code (Discovery):**
```ini
UserParameter=lvm.vg.discovery, sudo /usr/sbin/vgs --noheadings -o vg_name | sed -e 's/^[ \t]*//' | grep -v '^$' | awk '{printf "{\"{#VGNAME}\":\"%s\"}\n", $1}' | paste -sd, | sed -e 's/^/[/' -e 's/$/]/'
```
**How the Linux Code Works:**
- `vgs --noheadings -o vg_name`: Outputs just the raw names of the Volume Groups.
- `sed` and `grep`: Clean up invisible whitespaces and blank lines.
- `awk`: Wraps the Volume Group name perfectly into Zabbix JSON `{"{#VGNAME}":"rhel"}`.
- `paste` and `sed`: Joins them with commas and wraps the whole string in brackets `[]`.

**Agent Code (Metrics):**
```ini
UserParameter=lvm.vg.size[*], sudo /usr/sbin/vgs --noheadings --units b -o vg_size $1 | sed -e 's/^[ \t]*//' -e 's/B//'
```
**How the Linux Code Works:**
- `--units b -o vg_size $1`: Forces LVM to output the exact capacity in raw Bytes for the requested Volume Group (`$1`).
- `sed -e 's/B//'`: Strips the "B" character off the end so Zabbix receives a pure, graphable integer (e.g., converting `53687091200B` to `53687091200`).
