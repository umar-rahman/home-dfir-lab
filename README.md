# home-dfir-lab

Isolated home lab in VirtualBox: simulated network attacks against a vulnerable host, detected with Suricata and Zeek, investigated end-to-end and mapped to MITRE ATT&CK, with a full incident report.

> **Status:** ✅ Complete — full attack → detection → investigation → report chain documented.

---

## Overview

This project covers both sides of a network intrusion in a self-built, isolated lab:

1. **Attack:** stage an attack from Kali Linux against a deliberately vulnerable VM.
2. **Detect:** capture and alert on the activity using Suricata (IDS) and Zeek (network logging).
3. **Investigate:** reconstruct the timeline, root cause and impact from log and packet-capture evidence.
4. **Map:** align attacker activity to MITRE ATT&CK techniques.
5. **Report:** produce a professional incident report with prioritised remediation.

The aim is to build a practical understanding of how attacks unfold and what evidence they leave behind, from both the attacker's and the defender's point of view.

---

## Lab Environment

| Component | Details |
|---|---|
| Host | Windows, AMD Ryzen 7 7700, 32 GB RAM |
| Hypervisor | Oracle VirtualBox (Type 2) |
| Attacker | Kali Linux: 4 vCPUs, 4 GB RAM |
| Victim | Metasploitable 2 |
| Detection | Suricata, Zeek |

### Network design

```
                 ┌──────────────── VirtualBox ────────────────┐
  Internet ◄─NAT─┤ eth0  Kali Linux (attacker)                 │
                 │ eth1 ──┐                                    │
                 │        │  Internal Network (isolated)       │
                 │        └──── Metasploitable 2 (victim)      │
                 │              (no internet access)           │
                 └─────────────────────────────────────────────┘
```

- **NAT adapter (Kali only):** outbound internet for updates and tooling.
- **Internal Network:** carries all attack traffic between Kali and the victim. It is not reachable from the host or home LAN, and the vulnerable VM never gets internet access.

---

## Progress

- [x] Install VirtualBox
- [x] Import and boot Kali Linux (NAT)
- [x] Verify networking (`ip a`, `ip route`)
- [x] Add metasploitable VM and configure the isolated internal network
- [x] Confirm network matches the design
- [x] Stage the attack
- [x] Detect and triage alerts
- [x] Investigate: timeline, root cause, impact
- [x] Map to MITRE ATT&CK
- [x] Write the incident report

---

## Documentation

| Stage | Document |
|---|---|
| 1. Lab setup | [docs/01-lab-setup.md](docs/01-lab-setup.md) |
| 2. Recon and exploitation | [docs/02-recon-and-exploitation.md](docs/02-recon-and-exploitation.md) |
| 3. Detection | [docs/03-detection.md](docs/03-detection.md) |
| 4. Investigation | Covered in the [incident report](report/incident-report.md) (timeline, root cause, impact) |
| 5. ATT&CK mapping | Covered in the [incident report](report/incident-report.md) (§8) |
| 6. Incident report | [report/incident-report.md](report/incident-report.md) |

---

## Repository Structure

```
home-dfir-lab/
├── README.md
├── docs/                 # Write-ups for each stage
├── evidence/
│   └── screenshots/      # Command output and tool screenshots
├── configs/              # Suricata rules and Zeek scripts
└── report/               # Final incident report
```

---

## Tools

VirtualBox · Kali Linux · Metasploitable 2 · Suricata · Zeek · Wireshark · MITRE ATT&CK

---

## Scope & Ethics

All activity is performed in an isolated VirtualBox lab against intentionally vulnerable virtual machines that I own and control. No external systems or networks are targeted.

---

## Lessons Learned

- **Signature-based detection only catches what it has a rule for.** Suricata flagged the noisy Nmap scan immediately, but the actual break-in walked straight past the ~53,000 ET Open rules because none matched the vsftpd backdoor trigger. Writing a custom rule (SID 1000001) to detect it, and proving it fired, was the most valuable part of the project. Detection engineering is noticing the gap and closing it, not just running the tool.

- **An IDS only sees live traffic on the interface it is watching.** My first recon scans happened before Suricata was started, so they were never recorded. The sensor has to be running, and placed where the traffic actually crosses. This is also why I was explicit that detection ran on the attacker host for simplicity, where a real deployment would use a dedicated sensor with traffic mirroring.

- **Match the tool to the scale.** Zeek produces structured, searchable logs that are powerful across high-volume production traffic, but in a single-host lab Wireshark alone was enough to analyse the handful of connections involved. I included Zeek to demonstrate the network-security-monitoring approach, while being honest about where its real value shows.

- **Isolation is a design decision, not an afterthought.** "No route out at all" (an Internal Network with no gateway) is stronger than "a route that happens to be firewalled." Testing it from the victim's side (ping the internet and expect it to fail, ping Kali and expect it to succeed) proved the isolation rather than assuming it.

- **Doing the exploit by hand teaches more than automating it.** Triggering the backdoor with netcat over two separate connections showed exactly what happens on the wire, the FTP commands involved, and why the shell is fragile, all of which Metasploit's one-click module hides.

- **Keep software updated, and use terms precisely.** An outdated version running a publicly documented exploit is trivially compromised. I also learned the distinction between an outdated version of still-maintained software (vsftpd 2.3.4, which simply needs updating to the current 3.x line) and genuinely end-of-life software (a retired product with no supported version at all). The two call for different remediations.

---

## Author

**Umar Rahman** · University of Southampton
