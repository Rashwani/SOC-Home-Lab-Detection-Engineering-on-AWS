# SOC-Home-Lab-Detection-Engineering-on-AWS

A self-built Security Operations Center lab simulating a full detection pipeline &mdash; adversary simulation, endpoint telemetry, SIEM ingestion, alert triage, and incident reporting &mdash; built on a free-tier AWS account.
 
**Stack:** AWS EC2 &middot; Wazuh (SIEM) &middot; Sysmon &middot; MITRE Caldera &middot; Atomic Red Team
 
---
 
## Architecture
 
Three EC2 instances inside an isolated VPC, communicating over private IPs:
 
| VM | OS | Role |
|---|---|---|
| Wazuh Server | Ubuntu 24.04 LTS | SIEM manager, indexer, dashboard |
| Windows Endpoint | Windows Server 2022 | Monitored "victim" machine, Sysmon + Wazuh agent |
| Attack Machine | Ubuntu 24.04 LTS | Runs Caldera C2 + Atomic Red Team |
 
![VPC resource map](screenshots/03-vpc-resource-map.png)
*Custom VPC with public/private subnets across two availability zones.*
 
![Security group inbound rules](screenshots/02-security-group-rules.png)
*Locked-down security group &mdash; only SSH, RDP, the Wazuh dashboard, agent communication, and Caldera's UI are exposed, each scoped to my IP or the internal VPC range.*
 
![Three EC2 instances running](screenshots/04-ec2-instances-running.png)
*All three lab VMs healthy and passing status checks.*
 
---
 
## SIEM Setup
 
Wazuh was installed as the all-in-one manager/indexer/dashboard, with agents deployed to both the Windows Endpoint and the Attack Machine.
 
![Wazuh agents active](screenshots/05-wazuh-agents-active.png)
*Both agents reporting Active with 100% coverage.*
 
![Wazuh modules dashboard](screenshots/06-wazuh-modules-dashboard.png)
*Wazuh's module overview &mdash; Security Events, MITRE ATT&CK mapping, and compliance modules (PCI DSS, NIST 800-53) all active out of the box.*
 
Sysmon was installed on the Windows Endpoint (SwiftOnSecurity config) and wired into the Wazuh agent's `ossec.conf` as an additional log source, giving Wazuh visibility into detailed process creation, network, and file events beyond default Windows Event Logs.
 
---
 
## Adversary Emulation
 
Two complementary attack simulation tools were used to generate realistic telemetry:
 
**MITRE Caldera** &mdash; automated, multi-step adversary emulation via a deployed Sandcat agent.
 
![Caldera agent connected](screenshots/07-caldera-agent-connected.png)
*Sandcat agent live on the Windows Endpoint, elevated privileges, ready to receive operations.*
 
**Atomic Red Team** &mdash; individual MITRE ATT&CK technique tests run directly against the endpoint, including encoded PowerShell execution (T1059.001) and account discovery (T1087) techniques.
 
---
 
## Detection Results
 
The detection pipeline (Sysmon &rarr; Wazuh agent &rarr; Wazuh rule engine) successfully identified and MITRE-mapped multiple attack techniques in real time.
 
### Case 1: Encoded PowerShell Execution (T1059.001)
 
![Wazuh alert for T1059.001](screenshots/01-wazuh-alert-t1059.png)
*High-severity (level 12) alert: PowerShell spawning a child process executing a Base64-encoded command &mdash; a common evasion technique.*
 
![Raw Sysmon event detail](screenshots/08-sysmon-raw-event-t1059.png)
*Full process lineage captured by Sysmon: parent PowerShell &rarr; cmd.exe &rarr; encoded PowerShell payload, with process hashes and user context.*
 
This alert was fully triaged and documented as a formal incident report (Detection, Investigation, MITRE Mapping, Verdict, Recommendation) &mdash; see [`incident-report-001.pdf`](incident-report-001.pdf).
 
### Case 2: Additional Techniques Detected
 
![Additional alerts T1105 and T1548.003](screenshots/09-additional-alerts-t1105-t1548.png)
*Further activity automatically classified by Wazuh, including T1105 (Ingress Tool Transfer &mdash; executable dropped in a folder commonly used by malware, level 15) and T1548.003 (privilege escalation via sudo).*
 
---
 
## Key Takeaways
 
- Built and hardened a segmented AWS VPC from scratch, with least-privilege security group rules
- Configured a production-style SIEM pipeline (Sysmon &rarr; Wazuh) and diagnosed real log-collection failures (misconfigured `ossec.conf`, missing log sources) along the way
- Ran both single-technique (Atomic Red Team) and multi-step chained (Caldera) adversary simulations
- Achieved real MITRE ATT&CK-mapped detections across multiple tactics: Execution, Discovery, Command and Control, Privilege Escalation
- Practiced SOC analyst workflow end-to-end: alert triage, classification (true positive / false positive / benign true positive), and formal incident reporting
---
 
## Repo Contents
 
- `screenshots/` &mdash; supporting evidence for each stage of the build
- `incident-report-001.pdf` &mdash; full incident report for the T1059.001 detection
---
 














