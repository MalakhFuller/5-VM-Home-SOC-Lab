# SOC Lab 2: 5x VMs | Wazuh SIEM | AD DC | 2 Victims | 1 AttackBox

This repository documents the build-out and exercises of a multi-VM SOC lab environment designed to simulate real enterprise security operations, built as a hands-on learning environment to develop SOC analyst and detection engineering skills. The lab features a Wazuh SIEM/XDR manager receiving telemetry from Windows and Linux endpoints instrumented with Sysmon, with a dedicated Kali Linux attack box for adversary simulation mapped to MITRE ATT&CK techniques. Each project builds on the last, progressively covering detection engineering, log analysis, threat hunting, and incident response workflows.

---

## Lab Environment

| VM | Role | OS | IP |
|---|---|---|---|
| Wazuh-Server | SIEM / XDR | Ubuntu Server 26.04 | 10.10.10.130 |
| Kali-AttackBox | Attack Simulation | Kali Linux 2026.1 | 10.10.10.128 |
| Win11-Victim01 | Windows Endpoint | Windows 11 Enterprise (Eval) | 10.10.10.131 |
| Ubuntu-Victim02 | Linux Endpoint | Ubuntu Server 26.04 | 10.10.10.132 |
| WinServer-DC01 | Active Directory DC | Windows Server 2025 (Eval) | 10.10.10.134 |

All VMs operate on an isolated internal network (VMnet2, 10.10.10.0/24) with no direct access to the host machine or real home network.

---

## Phase 1: Lab Build

### Project 1 — Full 5-VM SOC Lab Build | Wazuh SIEM | Active Directory | Sysmon

Complete build-out of a 5-VM isolated SOC lab from scratch, including hypervisor configuration, network segmentation, SIEM deployment, endpoint instrumentation, Active Directory setup, and domain joining of both Windows and Linux endpoints.

**Skills demonstrated:**
- Multi-VM hypervisor configuration (VMware Workstation Pro)
- Isolated network segmentation using custom VMnets
- SIEM deployment and configuration (Wazuh 4.14)
- Wazuh agent deployment on Windows and Linux endpoints
- Sysmon installation and configuration (SwiftOnSecurity ruleset)
- Active Directory domain deployment (forest, OUs, users, groups)
- Domain joining of Windows and Linux endpoints
- Log collection and forwarding configuration
- Basic alert triage — identifying false positives from admin activity

**Status:** [Completed](./01_SOC-Lab-2-Full-Build.md)

---

## Phase 2: Attack Simulation and Detection (Planned)

### Project 2 — Credential Attack Against Active Directory
Simulating a password spray attack against domain accounts using Kali, with detection via Wazuh (Event ID 4625, 4771) and MITRE ATT&CK mapping (T1110.003).

**Status:** Planned, but delayed while I wire a dual SIEM with Suricata running passive for better detection and analysis capabilities.

---

### Project 3 — Kerberoasting
Requesting service tickets for domain accounts from Kali and detecting the attack signature in Wazuh via Event ID 4769. MITRE ATT&CK mapping: T1558.003.

**Status:** Planned, but delayed while I wire a dual SIEM with Suricata running passive for better detection and analysis capabilities.

---

### Project 4 — Lateral Movement Detection
Using compromised credentials to move between endpoints, with Sysmon and Wazuh tracing the full process chain and kill path.

**Status:** Planned, but delayed while I wire a dual SIEM with Suricata running passive for better detection and analysis capabilities.

---

### Project 5 — Custom Detection Rule Writing
Identifying a gap in Wazuh's default detection coverage and writing a custom rule to close it — the core skill of detection engineering.

**Status:** Planned, but delayed while I wire a dual SIEM with Suricata running passive for better detection and analysis capabilities.

---

### Project 6 — Full Purple Team Exercise
End-to-end attack chain from initial reconnaissance through domain compromise, fully documented as a SOC incident report.

**Status:** Planned, but delayed while I wire a dual SIEM with Suricata running passive for better detection and analysis capabilities.

---

## Author

[Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)
