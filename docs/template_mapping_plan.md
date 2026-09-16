# Zabbix Template Mapping: Host Health & Availability

This document maps the exact requirements and thresholds from the spreadsheet to the Zabbix implementation plan.

## OBS-F-001: Linux Host State Monitoring
* **Requirement:** The monitoring system SHALL continuously monitor the operational state of each Linux server and detect transitions between running, degraded, unreachable and stopped states.
* **Threshold:** None specified (Log/State collection only)
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-002: System Lifecycle Event Logging
* **Requirement:** The monitoring system SHALL collect and retain operating system events related to start-up, shutdown, reboot, kernel panic and recovery.
* **Threshold:** None specified (Log/State collection only)
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-005: Unplanned Restart Detection
* **Requirement:** The monitoring system SHALL distinguish between planned and unplanned host restarts and generate an alert for any restart not associated with an approved maintenance activity.
* **Threshold:** None specified (Log/State collection only)
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-012: Boot Outcome Monitoring
* **Requirement:** The monitoring system SHALL monitor the outcome of the operating system boot sequence and detect services that failed to start following host initialisation.
* **Threshold:** None specified (Log/State collection only)
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-013: Kernel Error Monitoring
* **Requirement:** The monitoring system SHALL collect kernel diagnostic messages and detect kernel errors, hardware exceptions, filesystem errors and driver failures.
* **Threshold:** None specified (Log/State collection only)
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-003: Resource Utilisation Monitoring
* **Requirement:** The monitoring system SHALL monitor CPU, memory, swap, filesystem and I/O utilisation metrics on each Linux server.
* **Threshold Parameter:** Processor utilisation
* **Warning:** 80% sustained 5 min
* **Critical:** 95% sustained 5 min
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-003: Resource Utilisation Monitoring
* **Requirement:** The monitoring system SHALL monitor CPU, memory, swap, filesystem and I/O utilisation metrics on each Linux server.
* **Threshold Parameter:** Available memory
* **Warning:** below 20%
* **Critical:** below 10%
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-003: Resource Utilisation Monitoring
* **Requirement:** The monitoring system SHALL monitor CPU, memory, swap, filesystem and I/O utilisation metrics on each Linux server.
* **Threshold Parameter:** Swap activity rate
* **Warning:** any sustained 5 min
* **Critical:** —
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-003: Resource Utilisation Monitoring
* **Requirement:** The monitoring system SHALL monitor CPU, memory, swap, filesystem and I/O utilisation metrics on each Linux server.
* **Threshold Parameter:** Processor I/O wait
* **Warning:** 20% sustained 5 min
* **Critical:** 40% sustained 5 min
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-015: Memory Exhaustion Event Monitoring
* **Requirement:** The monitoring system SHALL detect and record kernel out-of-memory conditions, identifying the affected host and the terminated process.
* **Threshold:** None specified (Log/State collection only)
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-004: Service Status Monitoring
* **Requirement:** The monitoring system SHALL monitor configured operating system services and detect service start, stop, restart and failure events.
* **Threshold:** None specified (Log/State collection only)
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-016: Critical Process Monitoring
* **Requirement:** The monitoring system SHALL monitor the presence and resource consumption of processes designated as operationally critical, and detect absent processes and processes exceeding configured consumption limits.
* **Threshold Parameter:** Critical process processor use
* **Warning:** 80% of one core, 10 min
* **Critical:** —
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-016: Critical Process Monitoring
* **Requirement:** The monitoring system SHALL monitor the presence and resource consumption of processes designated as operationally critical, and detect absent processes and processes exceeding configured consumption limits.
* **Threshold Parameter:** Critical process memory use
* **Warning:** 80% of host memory
* **Critical:** —
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-017: Service Restart Loop Detection
* **Requirement:** The monitoring system SHALL detect services that restart repeatedly within a configurable observation period.
* **Threshold Parameter:** Service restart rate
* **Warning:** 3 restarts in 10 min
* **Critical:** 5 restarts in 10 min
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-018: File Descriptor Utilisation Monitoring
* **Requirement:** The monitoring system SHALL monitor open file descriptor counts at system and process level against configured limits.
* **Threshold Parameter:** File descriptor utilisation
* **Warning:** 70% of configured limit
* **Critical:** 85% of configured limit
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-019: Physical Hardware Health Monitoring
* **Requirement:** The monitoring system SHALL monitor hardware health indicators reported by physical hosts, including temperature, cooling component status, power supply status, redundancy, and memory error rates.
* **Threshold Parameter:** Hardware health indicators
* **Warning:** Per vendor specification
* **Critical:** Per vendor specification
* **Zabbix Implementation Strategy:** (To be determined)

## OBS-F-020: Hardware Event Log Collection
* **Requirement:** The monitoring system SHALL collect entries recorded within the platform management controller event log of each physical host.
* **Threshold:** None specified (Log/State collection only)
* **Zabbix Implementation Strategy:** (To be determined)

