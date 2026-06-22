# SOC Lab 2: 5x VMs | Wazuh SIEM | AD DC | 2 Victims | 1 AttackBox

This is where I'm learning to defend a network by first building one worth attacking.

It's a five-VM SOC lab, fully isolated: a Wazuh SIEM/XDR taking telemetry from Windows and Linux endpoints wired with Sysmon, an Active Directory domain to give the attacks something real to move through, and a Kali box to do the attacking — mapped to MITRE ATT&CK as I go. The work runs in order, each project building on the last, from "stand up the environment" toward "catch a full attack chain inside it." The writeups are the honest version: where I took a wrong turn or assumed something that wasn't true, it's left in, because that's the part that actually taught me something.

---

## Lab Environment

| VM | Role | OS | IP |
|---|---|---|---|
| Wazuh-Server | SIEM / XDR | Ubuntu Server 26.04 | 10.10.10.130 |
| Kali-AttackBox | Attack Simulation | Kali Linux 2026.1 | 10.10.10.128 |
| Win11-Victim01 | Windows Endpoint | Windows 11 Enterprise (Eval) | 10.10.10.131 |
| Ubuntu-Victim02 | Linux Endpoint | Ubuntu Server 26.04 | 10.10.10.133 |
| WinServer-DC01 | Active Directory DC | Windows Server 2025 (Eval) | 10.10.10.134 |

Everything lives on an isolated internal segment (VMnet2, 10.10.10.0/24) with no path to the host or my real home network. Internet is the exception, not the default — NAT gets toggled on only for the install steps that need it, then pulled back off.

---

## Phase 1: Lab Build

### Project 1 — Full 5-VM SOC Lab Build | Wazuh SIEM | Active Directory | Sysmon

Building the whole environment from nothing: hypervisor and network segmentation, the Wazuh SIEM, endpoint instrumentation with Sysmon, a from-scratch Active Directory domain, and joining both a Windows and a Linux endpoint to it. The short version of what it taught me — the security tooling was rarely the hard part; the plumbing, permissions, and DNS around it were, which is exactly the stuff that breaks at 2am in a real SOC.

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

## Phase 2: Attack Simulation and Detection — *paused on purpose*

Phase 2 is the offensive half: run real attacks against the domain and catch them in the SIEM. It's deliberately on hold, and the reason is a resequencing call I'd make again. Before I start hunting attacks, I'm widening the lens — standing up a second SIEM (Splunk) alongside Wazuh and a passive Suricata sensor on the wire, so I'm watching from both the endpoint *and* the network at once instead of trusting a single source. 

Further, I built the original lab on Wazuh, but I keep seeing Splunk in job specifications for SOC roles. I want hands-on experience with the tools and processes that Recruiters care about. I have limits, in that I don't have access to the funds to build a full enterprise enviornment to play with. That said, I want to see what I *can* do within my limits. 

Everything below is queued behind that work. Once I get Splunk wired in, Suricata reporting on the network side, and provision some attack-ready accounts, including a Kerberoastable service account to attack, I will then start again with the below:

### Project 2 — Credential Attack Against Active Directory
Password spray against domain accounts from Kali, detected in Wazuh (and now also Splunk) via Event IDs 4625 / 4771. MITRE ATT&CK: T1110.003.

### Project 3 — Kerberoasting
Requesting service tickets for domain accounts and catching the signature in Wazuh (and now also Splunk) via Event ID 4769. MITRE ATT&CK: T1558.003.

### Project 4 — Lateral Movement Detection
Using compromised credentials to move between endpoints, with Sysmon and Wazuh (and now also Splunk) tracing the full process chain and kill path.

### Project 5 — Custom Detection Rule Writing
Finding a gap in Wazuh (and now also Splunk)'s default coverage and writing a rule to close it — the core skill of detection engineering, and the one I'm building toward.

### Project 6 — Full Purple Team Exercise
The capstone: an end-to-end attack chain from recon through domain compromise, written up as a complete SOC incident report.

These efforts unpaused on the 7-VM dual SIEM can be viewed [here.](https://github.com/MalakhFuller/7-VM-Home-SOC-Lab)

---

## Author

[Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)
