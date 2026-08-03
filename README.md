
A self-built Security Operations Center lab simulating a full detection pipeline &mdash; adversary simulation, endpoint telemetry, SIEM ingestion, alert triage, and incident reporting &mdash; built entirely on a free-tier AWS account.
 
**Stack:** AWS EC2 &middot; Wazuh (SIEM) &middot; Sysmon &middot; MITRE Caldera &middot; Atomic Red Team
 
---
 
## Table of Contents
 
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Phase 1: Infrastructure](#phase-1-infrastructure)
4. [Phase 2: Wazuh Server Installation](#phase-2-wazuh-server-installation)
5. [Phase 3: Agent Deployment](#phase-3-agent-deployment)
6. [Phase 4: Sysmon &amp; Endpoint Telemetry](#phase-4-sysmon--endpoint-telemetry)
7. [Phase 5: Adversary Simulation &mdash; Atomic Red Team](#phase-5-adversary-simulation--atomic-red-team)
8. [Phase 6: Adversary Simulation &mdash; MITRE Caldera](#phase-6-adversary-simulation--mitre-caldera)
9. [Phase 7: Detection &amp; Triage](#phase-7-detection--triage)
10. [Phase 8: Incident Reporting](#phase-8-incident-reporting)
11. [Troubleshooting Log](#troubleshooting-log)
12. [Cost Management](#cost-management)
13. [Key Skills Demonstrated](#key-skills-demonstrated)
---
 
## Overview
 
This lab simulates the core workflow of a SOC analyst: an attacker runs techniques against a target machine, a SIEM ingests telemetry from that machine, the SIEM maps activity to MITRE ATT&CK, and the analyst triages and documents the results. The goal was hands-on, practical experience with the same tools used in real security operations centers, built from scratch on constrained free-tier infrastructure &mdash; including diagnosing and fixing every failure along the way rather than following a guide that worked perfectly the first time.
 
---
 
## Architecture
 
Three EC2 instances inside a custom, segmented VPC, communicating over private IPs:
 
| VM | OS | Role | Instance Type | Storage |
|---|---|---|---|---|
| Wazuh Server | Ubuntu 24.04 LTS | SIEM manager, indexer, dashboard | t3.small | 20 GB |
| Windows Endpoint | Windows Server 2022 Base | Monitored "victim" machine | t3.medium | 30 GB |
| Attack Machine | Ubuntu 24.04 LTS | Runs Caldera C2 + Atomic Red Team | t3.micro &rarr; t3.small (temporary) | 10 GB |
<img width="1582" height="275" alt="Screenshot 2026-08-03 173309" src="https://github.com/user-attachments/assets/20de8972-1915-4fb0-9fbe-9c902ee59c83" />

 
**Network:** Custom VPC (`SOC-Lab-VPC`, CIDR `10.0.0.0/16`) with public and private subnets across two availability zones (`us-east-1a`, `us-east-1b`). A single security group (`SOC-Lab-SG`) enforces least-privilege inbound access:
 
| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | My IP | Remote into Linux VMs |
| RDP | 3389 | My IP | Remote into Windows VM |
| Custom TCP | 443 | My IP | Wazuh web dashboard |
| Custom TCP | 1514&ndash;1515 | `10.0.0.0/16` | Wazuh agent&ndash;server communication |
| Custom TCP | 8888 | My IP | Caldera web UI |
| All traffic | All | `10.0.0.0/16` | Inter-VM communication (agent enrollment, Sandcat callbacks) |
 
All three VMs share a single key pair (`soc-lab-key.pem`) for SSH access and Windows password decryption.
<img width="1850" height="686" alt="Screenshot 2026-08-03 172941" src="https://github.com/user-attachments/assets/33e3837c-489d-4d66-bc82-8bc6ace5f836" />
<img width="1498" height="635" alt="Screenshot 2026-08-03 173252" src="https://github.com/user-attachments/assets/46a939bb-d0a9-4767-9cd5-0d554c83b1c9" />

 
---
 
## Phase 1: Infrastructure
 
1. Created `SOC-Lab-VPC` via the AWS "VPC and more" wizard, auto-generating public/private subnets and route tables across two AZs.
2. Created `SOC-Lab-SG` and added the inbound rules listed above, scoping SSH/RDP/dashboard/Caldera access to my own IP and reserving VPC-internal access for agent and inter-VM traffic.
3. Launched all three EC2 instances into the same VPC, public subnet, with auto-assigned public IPs and the shared key pair.
4. Connected via:
```bash
   ssh -i soc-lab-key.pem ubuntu@<public-ip>
```
   For the Windows Endpoint, retrieved the auto-generated Administrator password through **Get Windows Password** (decrypted with the same `.pem` key) and connected via RDP.
 
---
 
## Phase 2: Wazuh Server Installation
 
On the Wazuh Server VM:
 
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a -i
```
 
The `-i` (`--ignore-check`) flag was required because Wazuh 4.7's installer only recognizes Ubuntu 16.04&ndash;22.04 as "supported" out of the box &mdash; Ubuntu 24.04 works in practice but fails the installer's version check without this flag.
 
Installation completed in 5&ndash;10 minutes, printing generated dashboard credentials. Accessed the dashboard at `https://<wazuh-server-public-ip>` and confirmed login.
 
Retrieved the server's **private IP** from the EC2 console (required for agent enrollment in the next phase).
 <img width="1896" height="858" alt="Screenshot 2026-08-03 173352" src="https://github.com/user-attachments/assets/11bfd2db-05b5-439f-a047-d7714ff8aa7b" />

---
 
## Phase 3: Agent Deployment
 
Deployed via **Agents &rarr; Deploy new agent** in the Wazuh dashboard, generating OS-specific install commands pointed at the Wazuh Server's private IP.
 
**Windows Endpoint:** ran the generated PowerShell install command as Administrator, then:
```powershell
NET START Wazuh
```
 
**Attack Machine (Linux):** ran the generated DEB install commands, then:
```bash
sudo systemctl start wazuh-agent
```
 
Confirmed both agents showed **Active** in the dashboard's Agents view.
 <img width="1908" height="736" alt="Screenshot 2026-08-03 173338" src="https://github.com/user-attachments/assets/4f793412-c5cf-4298-bacb-b79526e025c2" />

---
 
## Phase 4: Sysmon & Endpoint Telemetry
 
Installed Sysmon on the Windows Endpoint using the SwiftOnSecurity community configuration for high-signal event coverage:
 
```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Sysmon.zip"
Expand-Archive C:\Sysmon.zip -DestinationPath C:\Sysmon
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Sysmon\sysmonconfig.xml"
C:\Sysmon\Sysmon64.exe -i C:\Sysmon\sysmonconfig.xml -accepteula
```
 
Verified installation and active logging:
```powershell
Get-Service Sysmon64
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```
 
Extended the Wazuh agent's `ossec.conf` to ingest the Sysmon event channel, adding this block inside `<ossec_config>` alongside the existing `<localfile>` entries:
 
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```
 
Restarted the agent to apply the change:
```powershell
Restart-Service Wazuh
```
 
Confirmed ingestion by checking the agent log for `Analyzing event log: 'Microsoft-Windows-Sysmon/Operational'` and cross-referencing fresh Sysmon events against Wazuh's Discover view.
 
---
 
## Phase 5: Adversary Simulation &mdash; Atomic Red Team
 
Installed the framework on the Windows Endpoint (Administrator PowerShell):
 
```powershell
Set-ExecutionPolicy Bypass -Scope CurrentUser -Force
$url = "https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1"
IEX (New-Object Net.WebClient).DownloadString($url)
Install-AtomicRedTeam -getAtomics
```
 
Ran a scoped, low-risk technique test:
```powershell
Invoke-AtomicTest T1059.001 -TestNumbers 17
```
 
This specific test (**T1059.001-17, "PowerShell Command Execution"**) was chosen deliberately over test 1 (a Mimikatz-based test) after confirming Windows Defender actively quarantines credential-dumping tooling &mdash; see [Troubleshooting Log](#troubleshooting-log).
 
To re-run tests in fresh PowerShell sessions, the module needed reloading each time:
```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```
 
---
 
## Phase 6: Adversary Simulation &mdash; MITRE Caldera
 
Installed on the Attack Machine (Apache Software Foundation's current repository, using a Python virtual environment to comply with Ubuntu 24.04's externally-managed-environment restriction):
 
```bash
sudo apt update && sudo apt install -y python3-pip python3-venv git
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
 
git clone https://github.com/apache/caldera.git --recursive
cd caldera
python3 -m venv .calderavenv
source .calderavenv/bin/activate
pip3 install -r requirements.txt
```
 
Installed Go (required for the Sandcat plugin to compile agent binaries and appear as a deploy option):
```bash
wget https://go.dev/dl/go1.24.4.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.24.4.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```
 
Started the server as a background process (survives SSH disconnects):
```bash
nohup python3 server.py --insecure --build > caldera.log 2>&1 &
```
 
Deployed a Sandcat agent to the Windows Endpoint via the dashboard's generated PowerShell command, targeting the Attack Machine's **private IP**, after excluding the payload path from Windows Defender:
```powershell
Add-MpPreference -ExclusionPath "C:\Users\Public"
```
 
Ran the built-in **Discovery** adversary profile (safe reconnaissance: `net user`, `net accounts`, admin share enumeration) against the connected agent via **Operations &rarr; Create**.
 
---
 
## Phase 7: Detection & Triage
 
Reviewed generated alerts in Wazuh's **Security Events** and **Discover** views, filtering by agent name and time range, and cross-referencing raw Sysmon event fields (`data.win.eventdata.commandLine`, `rule.mitre.id`, `rule.description`) against what was actually run.
 
### Confirmed Detections
 
| Technique | Tactic | Rule ID | Level | Description |
|---|---|---|---|---|
| T1059.001 | Execution | 92057 | 12 | PowerShell spawning a child process executing a Base64-encoded command |
| T1087 | Discovery | 92031 / 92039 | 3 | `net.exe`/`net1.exe` account discovery commands |
| T1105 | Command and Control | 92213 | 15 | Executable dropped in a folder commonly used by malware |
| T1548.003 | Privilege Escalation | 5402 | 3 | Successful `sudo` to root |
 
Each alert was classified using standard SOC triage categories:
- **True Positive** &mdash; real (self-generated) attack activity, correctly detected
- **False Positive** &mdash; benign activity incorrectly flagged (e.g. AWS's own `EC2Launch.exe` wallpaper script was initially mistaken for suspicious `cmd.exe` activity until the command line was inspected directly)
- **Benign True Positive** &mdash; legitimate activity, correctly detected as noteworthy but not malicious
  <img width="1863" height="67" alt="Screenshot 2026-08-03 172728" src="https://github.com/user-attachments/assets/4c8f014b-9873-4614-aa7d-4e398fbb1b5f" />
  <img width="1736" height="672" alt="Screenshot 2026-08-03 174110" src="https://github.com/user-attachments/assets/bf4b85e4-f2e0-47a9-91d5-039d0e77e3ed" />
  <img width="1858" height="737" alt="Screenshot 2026-08-03 174206" src="https://github.com/user-attachments/assets/9c1e13f1-8959-482d-9ef0-9888c31214b4" />
---
 
## Phase 8: Incident Reporting
 
Documented the T1059.001 detection as a formal incident report following standard SOC structure:
 
1. **Detection** &mdash; alert source, rule ID, severity, MITRE mapping
2. **Investigation** &mdash; full process lineage (parent PowerShell &rarr; cmd.exe &rarr; encoded child PowerShell), decoded payload context, supporting host/file metadata (hashes, integrity level, working directory)
3. **MITRE ATT&CK Mapping** &mdash; Tactic: Execution (TA0002); Technique: T1059 Command and Scripting Interpreter; Sub-technique: T1059.001 PowerShell
4. **Verdict** &mdash; True Positive (self-generated Atomic Red Team test)
5. **Recommendation** &mdash; containment steps appropriate for a production environment (host isolation, payload decoding, process ancestry review, PowerShell Script Block Logging as a defense-in-depth addition)
---
 
## Troubleshooting Log
 
Real infrastructure work rarely goes smoothly on the first pass. These were the concrete issues diagnosed and resolved during the build:
 
**AMI selection pitfalls.** Initial instance launches repeatedly surfaced Windows/SQL Server bundled AMIs and "Ubuntu Pro" variants when searching by OS name alone. Both cause problems: SQL Server bundles enforce minimum instance-size licensing requirements (blocking smaller free-tier-friendly types), and Ubuntu Pro adds unnecessary paid-support complexity. Resolved by searching exact AMI name patterns (`Windows_Server-2022-English-Full-Base`, plain `Ubuntu Server 24.04 LTS`) and verifying "Free tier eligible" / publisher fields before launching.
 
**Free-tier vCPU quota limits.** Attempting to launch `t3.medium` for the Windows Endpoint was blocked by a default low EC2 vCPU service quota (unrelated to billing &mdash; a fraud-prevention default on new accounts). Resolved by requesting a Service Quotas increase for "Running On-Demand Standard instances," which was auto-approved, while launching with `t3.small` in the interim.
 
**Silent XML config failures.** Two separate config errors caused the Sysmon log source to be silently ignored by the Wazuh agent rather than throwing an obvious error:
- A missing closing `</localfile>` tag on the preceding `active-response` block left the file structurally invalid.
- The new Sysmon `<localfile>` block was initially appended *after* the closing `</ossec_config>` tag rather than inside it.
Both were diagnosed by checking `Get-WinEvent` locally (confirming Sysmon itself was logging correctly) against Wazuh's `Discover` view (showing no corresponding indexed events) &mdash; isolating the failure to the collection/config layer rather than the source.
 
**Self-referencing agent configuration.** An agent's `ossec.conf` was found pointing at its own private IP under `<server><address>` instead of the Wazuh Server's IP, causing repeated `(1208): Unable to connect to enrollment service` errors in `ossec.log`. Diagnosed by comparing the configured address against the SSH session's own hostname/prompt.
 
**Antivirus interference with red-team tooling.** Windows Defender actively quarantined both the Mimikatz-based Atomic Red Team test (T1059.001-1) and the Caldera Sandcat agent payload (`splunkd.exe`) within seconds of execution, surfacing as generic `"Access is denied"` PowerShell errors rather than an explicit antivirus message. Confirmed via `Get-MpThreatDetection`. Resolved with targeted `Add-MpPreference -ExclusionPath` exclusions rather than disabling Defender outright, preserving realistic endpoint defenses for the rest of the environment while unblocking specific, intentional test payloads.
 
**Resource-constrained builds.** Compiling Caldera's Vue.js frontend (`vite build`) on a `t3.micro` instance (1 GB RAM) caused severe memory pressure and an unresponsive SSH session. Resolved by adding a 2 GB swap file and temporarily resizing the instance to `t3.small` for the build step, then reverting to `t3.micro` afterward since the already-built frontend files don't require ongoing compute.
 
**Ubuntu 24.04 vs. tool version assumptions.** Both the Wazuh installer (`-i` flag) and Caldera's Sandcat plugin (requiring a manually installed Go toolchain, since it wasn't present by default and apt's version lagged upstream) needed explicit handling to work correctly on a newer Ubuntu release than either tool's documentation defaulted to.
 
**Post-reboot state loss.** Public IPs, the Caldera server process (run via `nohup` in a now-closed session), and the Sandcat agent process all needed to be re-established after stopping/starting instances between lab sessions &mdash; a reminder that only EBS-backed disk state persists across a stop/start cycle, not running processes or dynamically assigned public IPs.
 
 
## Key Skills Demonstrated
 
- Designing and securing a segmented cloud network with least-privilege firewall rules
- Deploying and configuring a production-grade SIEM (Wazuh) end-to-end, including custom log source integration
- Root-cause diagnosis of real log-pipeline failures (config syntax, misdirected enrollment, silent ingestion gaps)
- Running both atomic (single-technique) and chained (multi-step) adversary emulations
- Interpreting and validating MITRE ATT&CK technique mapping against raw telemetry
- SOC analyst triage workflow: alert classification, investigation, and formal incident documentation
- Operating within real infrastructure constraints: free-tier quotas, resource-limited instances, and endpoint defenses actively working against red-team tooling
