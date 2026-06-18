# SOC Lab 2: 5x VMs | Wazuh SIEM | AD Domain Controller | 2 Victims | 1 AttackBox

**Completed:** 2026-06-01
**Author:** Malakh Fuller

> **Privacy note:** Internal lab IP addresses have been anonymized in this writeup and related screenshots. All testing was performed exclusively on my own isolated home lab network.

---

## Objective

The goal of this project was to build a multi-VM Home SOC Lab from scratch — completely isolated from my real home network — running entirely inside VMware Workstation Pro on a single host machine. The lab needed to include a SIEM, an attack platform, domain-joined Windows and Linux victim endpoints, and a functioning Active Directory domain controller.

This was a significant upgrade from my prior dual-machine lab setup, where I used a secondary laptop running Kali to attack my actual desktop. That arrangement worked for early learning but wasn't sustainable for anything more advanced, especially once malware simulation entered the picture. A fully contained VM-based environment was the right move.

---

## Tools and Technologies

| Category | Details |
|---|---|
| **Hypervisor** | VMware Workstation Pro (Broadcom, free for personal use) |
| **SIEM / XDR** | Wazuh 4.14 (Manager, Indexer, Dashboard) |
| **AttackBox** | Kali Linux 2026.1 |
| **Victim Endpoints** | Windows 11 Enterprise (Evaluation), Ubuntu Server 26.04 LTS |
| **Domain Controller** | Windows Server 2025 (Evaluation) |
| **Endpoint Tools** | Wazuh Agent 4.14.5, Sysmon v15.20 (SwiftOnSecurity config) |
| **Skills Applied** | PowerShell, Linux CLI, Active Directory, Network Segmentation, Log Forwarding |
| **Prior Knowledge** | CompTIA A+, Network+, Security+, CySA+ (in progress), TryHackMe SOC Level 1, prior home SOC lab |

---

## VM Roster

| VM | Role | RAM | vCPU | Disk | IP |
|---|---|---|---|---|---|
| Wazuh-Server | SIEM / XDR | 8GB | 4 | 200GB | 10.10.10.130 |
| Kali-AttackBox | Attack Simulation | 4GB | 2 | 80GB | 10.10.10.128 |
| Win11-Victim01 | Windows Endpoint + Wazuh Agent + Sysmon | 6GB | 2 | 100GB | 10.10.10.131 |
| WinServer-DC01 | Active Directory Domain Controller | 4GB | 2 | 80GB | 10.10.10.134 |
| Ubuntu-Victim02 | Linux Endpoint + Wazuh Agent | 2GB | 2 | 60GB | 10.10.10.132 |

---

## Network Architecture

All five VMs operate on an isolated internal network with no direct access to my real home network. VMnet8 (NAT) was used temporarily during installation steps requiring internet access, and removed afterward.

| Network | Purpose | Subnet |
|---|---|---|
| VMnet2 | SOC internal — all VMs | 10.10.10.0/24 |
| VMnet8 | NAT (temporary, for downloads only) | 192.168.XX.XX/24 |

---

## Summary

Completed a full 5-VM SOC lab deployment from scratch in a single focused build session, including hypervisor configuration, network segmentation, SIEM installation, endpoint instrumentation, Active Directory deployment, and domain joining of both Windows and Linux endpoints.

---

## Methodology

### 1. Install VMware Workstation Pro

This proved to be more difficult than expected — and the difficulty had nothing to do with the software itself. Navigating Broadcom's website after their acquisition of VMware was genuinely not the most user-friendly experience.

The first obstacle was landing on an older page that said the software was only free for 30 days. That didn't match what I had read — Broadcom announced on November 11, 2024 that VMware Workstation Pro would be free for commercial, educational, and personal users with no license key required. So something was off.

After a bit of clicking around, I created a profile on the Broadcom Support Portal. That seemed like the right move, but then I hit a wall — the page I landed on said "No data found." It turned out I had been dropped onto the "My Entitlements" page, which was empty because I hadn't purchased anything.

Eventually I found the actual download under **My Downloads → All Products**, searched for "VMware Workstation Pro", and it appeared at the bottom of a long list. After agreeing to the Terms and Conditions, the download finally became available. 274MB installer, clean install from there.

---

### 2. Create the Isolated Virtual Networks

Before touching a single VM, I needed the network infrastructure in place. Skipping this step and trying to fix it later is the kind of thing that causes headaches.

In VMware, I went to **Edit → Virtual Network Editor** and created two custom networks:

| Network | Purpose | Subnet |
|---|---|---|
| VMnet2 | SOC internal (all VMs) | 10.10.10.0/24 |
| VMnet3 | Originally planned for Kali isolation | 10.10.20.0/24 |

There was one non-obvious issue I ran into: VMnet2 and VMnet3 weren't showing up in the VM Network Adapter dropdown, even after creating them. After some troubleshooting, I found the fix — I needed to check **"Connect a host virtual adapter to this network"** for each custom network. Once that was checked and applied, both appeared in the dropdown as expected. In retrospect it makes sense, but it wasn't obvious from the UI.

I ultimately decided to put everything on VMnet2. The original plan was to put Kali on VMnet3 to simulate an attacker crossing a network segment boundary, but for this phase of the lab it made more sense to keep things simple. Real attackers don't always come from a separate segment anyway — what mattered was that Wazuh would see the traffic and generate alerts, not which virtual switch Kali was plugged into.

Final network layout:

| VM | Network |
|---|---|
| Kali-AttackBox | VMnet2 |
| Wazuh-Server | VMnet2 |
| Win11-Victim01 | VMnet2 |
| WinServer-DC01 | VMnet2 |
| Ubuntu-Victim02 | VMnet2 |

---

### 3. Set Up Kali-AttackBox

This was the easiest part of the build. A quick search led me to `https://www.kali.org/get-kali/#kali-virtual-machines`, where I downloaded the VMware pre-built image as a `.7z` archive. After extracting to `C:\VMs\Kali-AttackBox\`, I opened the `.vmx` file directly in VMware, adjusted the settings (4GB RAM, 2 cores, VMnet2), and powered it on.

Kali booted straight to the desktop. I ran `ip a` to confirm it had picked up an IP on the right subnet — it had (10.10.10.128) — and then took a snapshot named **"Fresh Install - Clean"**.

The default credentials for the pre-built Kali image are `kali` / `kali`.

---

### 4. Set Up the Wazuh Server (Ubuntu Server + Wazuh)

#### Phase 1 — Install Ubuntu Server

I downloaded Ubuntu Server 26.04 LTS from `https://ubuntu.com/download/server` and created a new VM: 8GB RAM, 4 cores, 200GB disk, VMnet2, pointed at the ISO.

The installation was largely straightforward. The network configuration screen only offered "Continue without network," which felt wrong, but after playing around there wasn't any other real path forward. The VM had already picked up a valid IP (10.10.10.130 via DHCP) — the installer was just being cautious. I continued without network and the install completed fine.

Key choices during install:
- Full disk, LVM, no encryption
- Username: `malakh`, Server name: `wazuh-server`
- OpenSSH: installed
- Ubuntu Pro: skipped

After rebooting and logging in, I added a second Network Adapter set to NAT (VMnet8) to give the server temporary internet access for the Wazuh installation. I confirmed both interfaces were up:

- `ens33` — 10.10.10.130 (VMnet2, lab network)
- `ens37` — 192.168.XX.XX (VMnet8, NAT/internet)

I ran `ping -c 3 google.com` to confirm internet connectivity before proceeding.

> **Note on `ping -c 3`:** The `-c 3` flag tells ping to stop after 3 packets instead of running indefinitely. Without it, you'd have to manually stop it with Ctrl+C. It's a habit worth building.

#### Phase 2 — Install Wazuh

I referenced the official Wazuh Quickstart guide at `https://documentation.wazuh.com/current/quickstart.html` for the install command. A rule I was actively working on building at the time: always get install commands from the official project documentation, not blog posts. Scripts often run as root, so source matters.

The current install command (as of the time of this writeup) is:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

Note: the URL path includes the specific version number (`4.14`). An earlier attempt with `4.x` returned an Access Denied error from the CDN — the generic path no longer works.

The installation took about 15 minutes and installed the Wazuh Manager, Indexer, and Dashboard on the same host. At the end it printed a summary with credentials:

```
User: admin
Password: [generated password]
```

Those credentials were saved immediately. I also ran the following to retrieve them from the install files:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

There was one additional wrinkle: the generated password didn't work on first login attempt, possibly due to a copy/paste character encoding issue. I reset it using the Wazuh password tool:

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p SOClab2026-Lab
```

After that, logging in at `https://10.10.10.130` worked as expected.

I then removed the NAT adapter and took a snapshot: **"Wazuh Installed - Clean"**.

---

### 5. Set Up Win11-Victim01

#### Phase 1 — Install Windows 11 Enterprise

Microsoft provides free 90-day evaluation copies of Windows 11 Enterprise, which is exactly what a lab like this called for. Downloaded from:

`https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise`

Two options were available: regular and LTSC. LTSC (Long Term Servicing Channel) is a stripped-down version built for specialized environments like ATMs and medical equipment. The regular Windows 11 Enterprise was the right choice for simulating a standard corporate endpoint.

VM settings: 6GB RAM, 1 socket / 2 cores, 100GB disk, VMnet2.

One note on the boot process: when the Boot Manager appeared, selecting "Boot normally" looped back to the same screen since there was no OS installed yet. The right choice was **"EFI VMware Virtual SATA CDROM Drive"** to boot from the ISO.

Setup choices worth noting:
- When asked about network: chose "I don't have internet" — that was how to avoid being forced into a Microsoft account and create a local account instead
- Privacy settings: turned everything off (no telemetry needed in a lab)
- Username: `Victim01`

Snapshot taken: **"Win11 Fresh Install - Clean"**

#### Phase 2 — Install VMware Tools

VMware Tools improves performance inside the VM and enables features like shared folders, which I needed for the Sysmon installation later.

Going to **VM → Install VMware Tools** from the menu mounted the installer. I found the `setup.exe` on the virtual CD drive inside the VM and ran it with the **Typical** installation option. After the install and reboot, shared folder functionality was available.

#### Phase 3 — Install Wazuh Agent

I temporarily added a NAT adapter for internet access, confirmed the VM had both IPs (`10.10.10.131` on VMnet2, `192.168.XX.XX` on NAT), then navigated to the Wazuh dashboard from inside the VM at `https://10.10.10.130`.

From the dashboard: **Deploy New Agent → Windows → MSI 32/64 bits**

Filled in:
- Server address: `10.10.10.130`
- Agent name: `Win11-Victim01`

The dashboard generated a PowerShell install command. I ran it in PowerShell as Administrator. The install pulled the MSI directly from Wazuh's servers and registered the agent with the SIEM automatically.

Started the agent service:
```powershell
NET START WazuhSvc
```

Back in the dashboard, Win11-Victim01 showed up immediately as **Active**. Wazuh had already started doing Vulnerability Detection — 13 CVEs in Microsoft Edge and Microsoft Teams appeared within minutes of connection, several rated High severity. That was real threat intelligence with essentially zero configuration on my part.

Removed the NAT adapter. Snapshot: **"Wazuh Agent Installed - Clean"**

#### Phase 4 — Install Sysmon

Sysmon (System Monitor) is a Windows system service that logs detailed process activity, network connections, file creation events, and more to the Windows Event Log. Without it, the visibility into what's actually happening on the endpoint is limited. With it, you get the kind of granular telemetry that makes SOC work actually meaningful.

Since the NAT adapter had been removed, I couldn't download Sysmon directly to the VM. I downloaded two files to the host machine instead:

- **Sysmon** (from Microsoft Sysinternals): `https://download.sysinternals.com/files/Sysmon.zip`
- **SwiftOnSecurity Sysmon config**: `https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml`

The SwiftOnSecurity config is a well-maintained community configuration that covers the most important Windows events for SOC detection use. It was a reasonable starting point before I started tuning my own rules.

I transferred both files to the VM using a VMware Shared Folder (configured under **VM → Settings → Options → Shared Folders**), then in PowerShell as Administrator:

```powershell
cd "\\vmware-host\Shared Folders\VMShare"
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Output confirmed:
- Configuration validated ✅
- Sysmon64 installed ✅
- SysmonDrv installed ✅
- Sysmon64 started ✅

Next, I needed to tell the Wazuh agent to collect Sysmon logs and forward them to the SIEM. That required editing the agent configuration file:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

I added the following block to the `<localfile>` section:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Then restarted the Wazuh agent:

```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

In the Wazuh dashboard under **Threat Hunting**, filtering for `rule.groups: sysmon`, events started flowing immediately. All of them were from my own setup activity — but that was expected, and actually instructive:

- *"Suspicious Windows cmd shell execution"*
- *"Windows command prompt started by an abnormal process"*
- *"PowerShell process created an executable file in Windows root folder"*
- *"PowerShell was used to delete files or directories"*
- *"SecEdit.exe binary in a suspicious location launched by PowerShell"*

These were all **false positives** — legitimate admin activity that looked suspicious to a SIEM. When I eventually ran real attacks from Kali, the same types of alerts would appear — but those would be **true positives**. Learning to tell the difference was one of the core skills I was actively working on.

Snapshot: **"Wazuh Agent + Sysmon Installed - Clean"**

---

### 6. Set Up Ubuntu-Victim02

The Ubuntu Server install process felt familiar at this point. New VM: 2GB RAM, 1 socket / 2 cores, 60GB disk, VMnet2. Pointed at the same Ubuntu Server ISO from earlier.

The install went cleanly — Ubuntu-Victim02 picked up `10.10.10.132` automatically on VMnet2. I installed OpenSSH, skipped Ubuntu Pro, and was at a working login prompt in a reasonable amount of time.

**Wazuh Agent Installation:**

Added a temporary NAT adapter, confirmed internet connectivity with `ping -c 3 google.com`, then installed the agent:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb \
  && sudo WAZUH_MANAGER='10.10.10.130' WAZUH_AGENT_NAME='Ubuntu-Victim02' \
  dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb
```

Started and enabled the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

Status: `active (running)`. Confirmed in the Wazuh dashboard — Ubuntu-Victim02 showed up as the second active agent.

Removed the NAT adapter. Snapshot: **"Wazuh Agent Installed - Clean"**

---

### 7. Set Up WinServer-DC01

This was the step I had been both looking forward to and mildly dreading. Active Directory is a significant piece of infrastructure and I knew going in that there would be more moving parts than the previous VMs.

#### Phase 1 — Install Windows Server 2025

Microsoft provides free 180-day evaluation copies of Windows Server from:

`https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025`

I chose **Windows Server 2025 Standard** (not Datacenter — that was for large enterprise environments with unlimited virtualization needs, overkill for a lab).

VM settings: 4GB RAM, 1 socket / 2 cores, 80GB disk, VMnet2.

The install was straightforward. Snapshot: **"Fresh Install - Clean"**

#### Phase 2 — Install Wazuh Agent

Same process as Win11-Victim01. Added a temporary NAT adapter, opened the Wazuh dashboard from a browser inside the VM, deployed a new agent configured for `WinServer-DC01`, ran the generated PowerShell command, and started the service with `NET START WazuhSvc`.

All three agents confirmed active in the Wazuh dashboard.

Removed the NAT adapter.

#### Phase 3 — Install Sysmon

Same process as Win11-Victim01, using the existing Shared Folder setup. Ran from PowerShell as Administrator:

```powershell
cd "\\vmware-host\Shared Folders\VMShare"
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Snapshot: **"Wazuh Agent + Sysmon Installed - Clean"**

#### Phase 4 — Install Active Directory Domain Services (AD DS)

Before getting into the steps, it was worth taking a moment to understand what AD DS actually is and why it matters.

At that point, WinServer-DC01 was just a standalone Windows machine — it only knew about itself and nothing else. Installing the AD DS role transforms it into a **directory service**: a central database that stores information about every object in the network (users, computers, groups, policies). Once promoted to a Domain Controller, every other machine that joins the domain authenticates through it. It becomes the central authority for the entire network.

In Server Manager: **Add Roles and Features → Role-based installation → Active Directory Domain Services → Add Features → Install**.

I checked "Restart automatically if required" and let it run.

#### Phase 5 — Promote to Domain Controller

After installation, a yellow flag appeared in Server Manager with a link to "Promote this server to a domain controller."

I selected **"Add a new forest"** — this was the first DC, so I was creating an entirely new domain structure from scratch.

A quick note on what a "forest" is, since it was new to me:

```
Forest
  └── Tree
        └── Domain
              └── Organizational Units (OUs)
                    └── Users, Computers, Groups
```

The **forest** is the outermost boundary — the entire AD environment lives inside it. For this lab, I created a single forest with a single domain: `soclab.local`. This is the most common setup in small-to-medium enterprises.

One issue I ran into: the prerequisites check failed because the local Administrator account had a blank password. AD refused to create a new domain with an unsecured Administrator account. Fixed it in PowerShell:

```powershell
net user Administrator [strong-password]
```

After rerunning the prerequisites check and hitting Install, the server promoted successfully and rebooted. Server Manager now showed **AD DS** and **DNS** as active roles.

Snapshot: **"DC Promotion Complete - Clean"**

#### Phase 6 — Create Organizational Units and Test Users

In Server Manager: **Tools → Active Directory Users and Computers**

Right-clicked `soclab.local` and created four Organizational Units (OUs):

- `Workstations`
- `Servers`
- `DomainUsers` (Note: "Users" was already taken by a default system container)
- `Groups`

Created two test users in the `DomainUsers` OU:

| Name | Username |
|---|---|
| John Smith | jsmith |
| Jane Doe | jdoe |

Both set with "Password never expires" and "User must not change password at next logon" unchecked — appropriate for a lab environment.

Created a security group `SOC-Lab-Users` in the `Groups` OU, then added jsmith and jdoe as members.

Snapshot: **"AD Structure Complete - Users and OUs Created"**

#### Phase 7 — Join Win11-Victim01 to the Domain

Before joining, I needed to point Win11-Victim01's DNS at the Domain Controller. Machines find the domain through DNS, so this step was critical.

In PowerShell on Win11-Victim01:

```powershell
netsh interface ip set dns "Ethernet0" static 10.10.10.134
nslookup soclab.local
```

The nslookup resolved to the correct address. Then:

```powershell
Add-Computer -DomainName soclab.local -Credential SOCLAB\Administrator -Restart
```

After the reboot, I confirmed the domain join:

```powershell
systeminfo | findstr /B /C:"Domain"
```

Output: `Domain: soclab.local`

All Wazuh agents confirmed still active. Snapshot: **"Joined soclab.local Domain - Clean"**

#### Phase 8 — Join Ubuntu-Victim02 to the Domain

Joining a Linux machine to a Windows AD domain requires some extra packages not installed by default. This was one of the more interesting steps in the whole build because it's something I didn't fully appreciate was possible before starting.

Added a temporary NAT adapter for internet access, then installed the required packages:

```bash
sudo apt update && sudo apt install -y realmd sssd sssd-tools adcli packagekit samba-common-bin oddjob oddjob-mkhomedir
```

What each package does:
- **realmd** — discovers and joins AD domains
- **sssd** — handles AD authentication on Linux
- **adcli** — AD command line interface
- **samba-common-bin** — Samba utilities for Windows/Linux interoperability

Then I tried discovering the domain:

```bash
sudo realm discover soclab.local
```

Got "No such realm found." The issue was DNS — Ubuntu-Victim02 wasn't pointed at the Domain Controller for DNS resolution. Fixed it:

```bash
sudo nano /etc/resolv.conf
# Changed nameserver to 10.10.10.134
```

Ran `sudo realm discover soclab.local` again and it returned the full domain information. Then joined:

```bash
sudo realm join -U Administrator soclab.local
```

Confirmed with `realm list`:

```
soclab.local
  configured: kerberos-member
  domain-name: soclab.local
  login-formats: %U@soclab.local
  login-policy: allow-realm-logins
```

Both Win11-Victim01 and Ubuntu-Victim02 appeared in the `Computers` container in Active Directory Users and Computers on WinServer-DC01.

Removed the NAT adapter. Snapshot: **"Joined soclab.local Domain - Clean"**

---

## Final Lab State

| VM | IP | Role | Domain Member | Wazuh Agent | Sysmon |
|---|---|---|---|---|---|
| Kali-AttackBox | 10.10.10.128 | Attacker | No | No | No |
| Wazuh-Server | 10.10.10.130 | SIEM / XDR | No | No | No |
| Win11-Victim01 | 10.10.10.131 | Windows Endpoint | Yes (soclab.local) | Yes | Yes |
| Ubuntu-Victim02 | 10.10.10.132 | Linux Endpoint | Yes (soclab.local) | Yes | No |
| WinServer-DC01 | 10.10.10.134 | Domain Controller | Yes (soclab.local) | Yes | Yes |

---

## Key Findings

### Finding 1 — Wazuh Begins Detecting Vulnerabilities Immediately

Within minutes of the Wazuh agent connecting on Win11-Victim01, the dashboard surfaced 13 CVEs — several rated High severity — in Microsoft Edge and Microsoft Teams. No manual configuration required. This was a useful early demonstration of what agent-based SIEM coverage looks like in practice: the endpoint starts reporting its posture immediately, and the SIEM starts correlating against known vulnerability data automatically.

### Finding 2 — Sysmon Flags All Admin Activity as Suspicious

After installing Sysmon and configuring the Wazuh agent to collect its logs, the Threat Hunting view immediately populated with events — all generated by my own setup work. Commands run in PowerShell as Administrator, file operations, process chains from the Wazuh agent installer — all flagged. This was my first hands-on experience with **false positives**: legitimate admin activity that looks suspicious to a SIEM. Learning to identify and tune these out is one of the foundational skills in SOC work, and seeing it happen on my own lab gave me a concrete frame of reference for why alert tuning matters.

### Finding 3 — Linux Can Join a Windows AD Domain, But DNS Must Come First

Joining Ubuntu-Victim02 to `soclab.local` required the `realmd` and `sssd` package stack, which is not installed by default. More importantly, the domain join failed until I manually pointed Ubuntu's DNS resolver at the Domain Controller. The `realm discover` command returned "No such realm found" until DNS was corrected — at which point the entire process worked cleanly. The lesson: in a mixed-OS environment, DNS is the foundation everything else depends on.

### Finding 4 — VMware Custom Networks Require a Non-Obvious Setting to Appear in VM Dropdowns

Custom VMnets created in the Virtual Network Editor don't automatically appear in the VM Network Adapter dropdown. The fix was checking **"Connect a host virtual adapter to this network"** for each custom VMnet. Without this, the networks exist on paper but the VMs can't select them. This wasn't documented anywhere obvious — it required troubleshooting to discover.

---

## Key Competencies Demonstrated

- Multi-VM hypervisor configuration (VMware Workstation Pro)
- Isolated network segmentation using custom VMnets
- SIEM deployment and configuration (Wazuh 4.14 — Manager, Indexer, Dashboard)
- Wazuh agent deployment on Windows and Linux endpoints
- Sysmon installation and configuration using community best-practice ruleset
- Log collection and forwarding configuration (ossec.conf)
- Active Directory domain deployment (AD DS role, forest creation, OU structure)
- Domain joining of both Windows and Linux endpoints
- Basic alert triage — identifying false positives from admin activity
- Snapshot management and systematic documentation throughout

---

## Employer-Relevant Skills

**Tools:** Wazuh, Sysmon, VMware Workstation Pro, PowerShell, Linux CLI (Ubuntu), Active Directory Users and Computers, Windows Server Manager

**Concepts:** SIEM architecture, endpoint instrumentation, agent-based log collection, network segmentation, vulnerability detection, Active Directory administration, false positive identification

**Frameworks:** MITRE ATT&CK (referenced for Phase 2 attack simulation planning)

---

## SOC Relevance

This lab mirrors the kind of environment a Tier 1 or Tier 2 SOC analyst operates in at an MSSP or enterprise security team. The core infrastructure — a SIEM receiving telemetry from instrumented endpoints across a domain — is the foundation of almost every SOC engagement.

The hands-on experience of configuring log collection, watching alerts fire in real time, and distinguishing legitimate admin activity from genuinely suspicious behavior is directly relevant to day-one SOC analyst work. Understanding how the data gets from the endpoint to the SIEM, and why certain events trigger alerts while others don't, is the kind of knowledge that makes alert triage faster and more accurate.

---

## HUMINT to SOC Translation

My background in HUMINT involves building operational frameworks from scratch, making decisions under uncertainty, and working through problems methodically when the documentation doesn't give you a clear path forward. That mindset showed up repeatedly throughout this build — from troubleshooting the VMnet dropdown issue, to tracking down the correct Wazuh package URL, to figuring out why Ubuntu couldn't discover the domain.

More specifically, the HUMINT skills of source verification and adversarial thinking translate directly. Always getting install commands from official documentation rather than random blog posts is the same instinct as verifying a source before acting on intelligence. And building detection logic requires thinking like an adversary — which is something HUMINT practitioners do by default.

---

## Plans for Phase 2

My prior dual-machine home SOC lab covered foundational ground:

- **Project 1:** Detecting Network Reconnaissance Activity
- **Project 2:** Investigating Repeated Failed Login Attempts
- **Project 3:** Investigating Suspicious PowerShell Activity with Sysmon
- **Project 4:** SOC-Style Phishing Email Triage Report
- **Project 5:** End-to-End SOC Investigation: Reconnaissance to Detection

With the 5x VM lab and Active Directory now in place, Phase 2 is planned to go significantly deeper:

- **Project 1:** Credential Attack Against Active Directory (password spray, MITRE T1110.003)
- **Project 2:** Kerberoasting (Event ID 4769 detection, MITRE T1558.003)
- **Project 3:** Lateral Movement Detection
- **Project 4:** Custom Detection Rule Writing
- **Project 5:** Full Purple Team Exercise — end-to-end attack chain with incident report

---

## Author

[Malakh Fuller](https://www.linkedin.com/in/malakhfuller/)
