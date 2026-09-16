# SOC Lab Project 4 — SIEM Monitoring with Wazuh

**Author:** Oluwaferanmi (Feranmi) Olayinka — MSc Cyber Security, De Montfort University Leicester
**Part of a 5-project SOC analyst home lab portfolio** — see Project 1 ([Linux/Apache](#)), Project 2 ([Network Traffic Analysis](#)), and Project 3 ([Active Directory](#)).

## Overview

This project stands up a Wazuh SIEM (Security Information and Event Management platform) and uses it to monitor the Active Directory lab built in Project 3, plus the Linux webserver from Project 1. The goal was to go beyond building isolated systems and start doing what a SOC actually does: centralize logs, generate alerts, map activity to MITRE ATT&CK, and investigate.

A dedicated Wazuh Manager VM was built from scratch for this project rather than reusing an existing machine, to keep the portfolio's architecture honest — each project's role in the lab stays clear and independently verifiable.

## Architecture

All VMs run on VMware Workstation Pro, on a Host-only virtual network (`192.168.200.0/24`):

| VM | Role | OS | IP Address |
|---|---|---|---|
| `Wazuh-Manager` | SIEM (indexer, manager, dashboard) | Ubuntu Server | 192.168.200.132 |
| `soc-lab-dc01` | Domain Controller for `soclab.local` (monitored) | Windows Server 2022 | 192.168.200.10 |
| `soc-lab-client01` | Domain-joined workstation (monitored) | Windows 11 Pro | 192.168.200.20 |
| `soc-lab-webserver` | Linux web server from Project 1 (monitored) | Ubuntu Server | 192.168.200.130 |

**Screenshot:** `screenshots/01-topology-diagram.png` — network/topology diagram showing the four VMs and the manager-agent relationships.

## Tools & Technologies

- Wazuh 4.14 (indexer, manager, dashboard)
- Wazuh agent (Windows and Linux)
- Sysmon (Sysinternals), configured with the SwiftOnSecurity community baseline
- VMware Workstation Pro, Host-only networking
- PowerShell (attack simulation)
- MITRE ATT&CK framework (alert mapping)

## Build Process

### 1. Topology and VM specs

Planned the topology above and provisioned the Wazuh Manager VM (Ubuntu Server, 192.168.200.132) on the same Host-only network as the existing AD lab.

**Screenshot:** `screenshots/02-vm-specs.png`

### 2–3. Wazuh Manager, Indexer, and Dashboard installation

Installed the Wazuh indexer, manager, and dashboard on the new VM and confirmed all services running and the dashboard reachable over HTTPS.

**Screenshot:** `screenshots/03-wazuh-services-running.png`
**Screenshot:** `screenshots/04-wazuh-dashboard-login.png`

### 4–5. Agent deployment

Deployed the Wazuh agent to all three monitored machines using the dashboard's "Deploy new agent" wizard, and confirmed all three showing **Active** status.

**Screenshot:** `screenshots/05-agents-active-dashboard.png`

### 6. Windows Event Log collection

Confirmed the Wazuh Windows agent ships with Application, Security, and System event log collection (via `eventchannel`) enabled by default in `ossec.conf` — no manual configuration required.

**Screenshot:** `screenshots/06-ossec-conf-eventchannel.png`

### 7. Test alert generation and MITRE ATT&CK mapping

Generated repeated failed domain logon attempts against `soc-lab-dc01` to validate the detection pipeline end-to-end. Wazuh rule **60122** ("Logon Failure - Unknown user or bad password", level 5) fired and was visible in the Threat Hunting module, mapped to MITRE ATT&CK technique **T1531** (Account Access Removal, tactic: Impact).

**Screenshot:** `screenshots/07-rule-60122-failed-logon-mitre.png`

### 8. Sysmon integration and custom detection rule

Installed Sysmon on `soc-lab-dc01` using the SwiftOnSecurity community baseline configuration, and added a `localfile` block to the Wazuh agent config to collect the `Microsoft-Windows-Sysmon/Operational` event channel.

**Screenshot:** `screenshots/08-sysmon-installation.png`

Wazuh's built-in Sysmon ruleset (the `92000` range) picked this up immediately and began firing on real process, file, and network activity.

**Screenshot:** `screenshots/09-sysmon-events-flowing.png`

To simulate a common attacker technique, I ran a PowerShell command with **`-EncodedCommand`** (base64-obfuscated execution — MITRE **T1059.001**, PowerShell, and **T1027**, Obfuscated Files or Information):

```powershell
$command = 'Write-Host "Test encoded command execution"'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($command)
$encoded = [Convert]::ToBase64String($bytes)
powershell.exe -EncodedCommand $encoded
```

Wazuh's built-in ruleset caught this immediately: rule **92057** — *"Powershell.exe spawned a powershell process which executed a base64 encoded command"* (level 12) — fired with MITRE mapping already included.

**Screenshot:** `screenshots/10-rule-92057-encoded-powershell-alert.png`
**Screenshot:** `screenshots/11-commandline-field-detail.png`

**Custom rule — a debugging case study.** I also tried writing my own local detection rule (`local_rules.xml`, rule ID `100010`) targeting the same technique, to go a step beyond the built-in ruleset. This turned into a genuinely useful troubleshooting exercise:

- **Bug 1 — wrong field path.** My first version referenced `win.eventdata.commandline` and guessed at `if_sid 92000` as a parent rule. Inspecting a real alert's raw JSON in the dashboard (below) showed that Wazuh actually nests decoded Windows event fields under `data.win.eventdata.*` and `data.win.system.*` — the `data.` prefix was missing, so the field never matched anything.

**Screenshot:** `screenshots/12-decoded-field-discovery.png` — the raw event JSON that revealed the real field structure
**Screenshot:** `screenshots/13-custom-rule-first-draft.png` — the original (buggy) rule

- **Bug 2 — missing rule chaining.** After fixing the field path, the rule still didn't fire. The rule had no `<if_sid>` or `<if_group>` tying it into Wazuh's rule evaluation tree, so it was never being evaluated against incoming events at all, regardless of whether its conditions were correct. Adding `<if_group>sysmon</if_group>` was needed to chain it in.

**Screenshot:** `screenshots/14-custom-rule-final-with-ifgroup.png` — the corrected rule

As of writing, the rule is still being refined — a good reminder that a SIEM's value comes as much from understanding *why* a rule doesn't fire as from writing the rule itself. Importantly, this didn't leave a detection gap: Wazuh's built-in Sysmon ruleset (rule 92057, above) already provides working, confirmed coverage of the same technique with MITRE mapping.

### 9. Documentation

This README.

## Challenges & Troubleshooting

- **VMnet mismatch (NAT vs. Host-only).** The Wazuh Manager and webserver VMs initially sat on VMware's NAT network (VMnet8) while the rest of the lab used Host-only (VMnet1), breaking agent enrollment. Fixed by moving both VMs' network adapters to VMnet1 and re-addressing them statically on the lab's `192.168.200.x` convention.
- **Host-to-dashboard connectivity.** After the manager moved to VMnet1, the host machine couldn't reach the dashboard because its VMnet1 adapter sat on a different subnet than the manually-addressed lab VMs. Fixed by adding a secondary IP to the host's VMware virtual adapter.
- **Static IP drift after adapter changes.** Moving a VM's network adapter to a different virtual switch causes it to pick up a new DHCP address on that switch's own subnet. Fixed with static IP configuration (netplan on Linux) after each adapter change.
- **Wazuh decoder field paths.** Windows event data decoded by Wazuh is nested under `data.win.*`, not `win.*` as shown in some older online examples — confirmed by inspecting raw alert JSON directly rather than trusting third-party rule snippets.
- **Custom rule chaining.** A Wazuh rule built purely from `<field>` conditions needs an `<if_sid>` or `<if_group>` to be evaluated at all — a rule can be syntactically perfect and still never run.

## Key Takeaways / Skills Demonstrated

- Deploying and configuring a production-style SIEM (Wazuh indexer/manager/dashboard) from scratch
- Agent-based log collection across mixed Windows and Linux environments
- Windows Event Log and Sysmon integration for endpoint visibility
- Reading and interpreting raw decoded event JSON to debug detection rules
- Mapping detected activity to the MITRE ATT&CK framework
- Systematic network troubleshooting across multiple virtual switches
- Writing (and debugging) custom Wazuh detection rules

## Repository Structure

```
soc-lab-4-wazuh-siem/
├── README.md
└── screenshots/
    ├── 01-topology-diagram.png
    ├── 02-vm-specs.png
    ├── 03-wazuh-services-running.png
    ├── 04-wazuh-dashboard-login.png
    ├── 05-agents-active-dashboard.png
    ├── 06-ossec-conf-eventchannel.png
    ├── 07-rule-60122-failed-logon-mitre.png
    ├── 08-sysmon-installation.png
    ├── 09-sysmon-events-flowing.png
    ├── 10-rule-92057-encoded-powershell-alert.png
    ├── 11-commandline-field-detail.png
    ├── 12-decoded-field-discovery.png
    ├── 13-custom-rule-first-draft.png
    └── 14-custom-rule-final-with-ifgroup.png
```
