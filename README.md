# Attack & Detect Homelab

A self-built home lab demonstrating offensive and defensive security skills: an attacker VM, a monitored target, and a SIEM that detects and alerts on malicious activity.

## Overview

This project simulates a small corporate network to practice the full attack-to-detection pipeline: generate an attack, confirm it's logged at the endpoint, confirm it reaches the SIEM, and confirm the SIEM correctly alerts on it. Built as a learning project while pursuing entry-level help desk and cybersecurity roles.

**Tools used:** VirtualBox, Kali Linux, Windows 11 Pro, Sysmon (SwiftOnSecurity config), Wazuh (SIEM)

## Architecture

| VM | Role | IP | Specs |
|---|---|---|---|
| Kali Linux | Attacker | 192.168.56.104 | 2 vCPU, 2-3 GB RAM |
| Windows 11 Pro | Target (Sysmon-instrumented) | 192.168.56.102 | 2 vCPU, 4 GB RAM |
| Wazuh (OVA) | SIEM — manager, indexer, dashboard | 192.168.56.103 | 2 vCPU, 4 GB RAM |

All VMs run on an isolated VirtualBox host-only network (`192.168.56.0/24`), with internet-facing (NAT) adapters disabled during attack exercises to keep traffic contained to the lab.

## Setup Summary

- Installed Sysmon on the Windows target using the [SwiftOnSecurity sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config) for high-signal process-creation logging.
- Deployed the Wazuh agent to Windows and forwarded the **Sysmon**, **Security**, **Application**, and **System** event channels to the Wazuh manager.
- Verified end-to-end log flow: Windows Event Log → Wazuh agent → Wazuh manager → rule engine → dashboard alert.

## Exercise 1: Reconnaissance

**Command:**
```
nmap -sV -T4 192.168.56.102
```

**Finding:** With default Windows Firewall rules, all 1000 scanned ports returned filtered/no-response — a good baseline showing default Windows hardening. Enabled Remote Desktop (port 3389) to simulate a commonly-exposed service, then re-scanned and confirmed Nmap correctly fingerprinted it as `ms-wbt-server`.

*Screenshot: `screenshots/01-nmap-filtered.png`, `screenshots/02-nmap-rdp-open.png`*

## Exercise 2: Brute-Force Detection

**Attempted:** Automated brute-force with Hydra (`hydra -l EmbraRoot -P rockyou.txt rdp://192.168.56.102`). Hydra's RDP module is explicitly flagged experimental, and it failed to establish connections against this target, likely an NLA/TLS negotiation incompatibility. Documenting tool limitations like this is a realistic part of security work.

**Pivoted to:** Manual repeated failed logins via `xfreerdp` with incorrect passwords, generating the same detection signal (Windows Event ID 4625) that an automated brute-force would produce.

**Result:**
- Windows Security log recorded Event ID 4625 ("Unknown user name or bad password"), Logon Type 3 (network), source `192.168.56.104`, target account `EmbraRoot`.
- Wazuh's Threat Hunting module surfaced this as an **Authentication Failure** alert — rule ID `60122`, level 5, fired 18 times.

**Observation:** Wazuh auto-mapped this alert to MITRE ATT&CK **T1531 (Account Access Removal)**. A more accurate technique for repeated failed logins is **T1110 (Brute Force)**. Flagging this kind of mapping gap is a useful analyst habit — SIEM auto-classification isn't always exactly right, and catching it is part of the job.

*Screenshot: `screenshots/03-event-4625.png`, `screenshots/04-wazuh-threat-hunting-alert.png`*

## Lessons Learned

- Windows Firewall blocks ICMP and most inbound ports by default — relying on `ping` alone for host discovery is unreliable; Nmap's other discovery methods (e.g., ARP on local subnets) can still find a "filtered" host.
- A SIEM's summary dashboards are often filtered to specific views — raw or categorized events can exist without showing up until you check the right module (in Wazuh's case, Threat Hunting rather than a generic search).
- VirtualBox VMs can experience clock drift; enabling "Hardware Clock in UTC Time" prevents timestamp mismatches between the attacker, target, and SIEM.
- Offensive tooling isn't always reliable out of the box (Hydra's RDP module here) — knowing a tool's limitations and pivoting to an alternate method is itself a practical skill.

## Next Steps

- Active Directory lab: domain controller, GPOs, and common AD attacks (Kerberoasting, password spraying)
- Vulnerable-target exercises (Metasploitable, DVWA, Juice Shop) with a full recon-to-report cycle
- Network-layer defense with pfSense/OPNsense and Suricata
- Small Python tools (log parser, port scanner) to support the above
