# home-dfir-lab

Isolated home lab in VirtualBox: simulated network attacks against a vulnerable host, detected with Suricata and Zeek, investigated end-to-end and mapped to MITRE ATT&CK, with a full incident report.

> **Status:** In progress. This README is updated as each stage is completed.

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
| Victim | *Planned:* Metasploitable 2 |
| Detection | *Planned:* Suricata, Zeek |

### Network design (planned)

```
                 ┌──────────────── VirtualBox ────────────────┐
  Internet ◄─NAT─┤ eth0  Kali Linux (attacker)                │
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
- [x] Change default credentials
- [x] Update Kali (`apt full-upgrade`) and take a clean baseline snapshot
- [ ] Add victim VM and configure the isolated internal network
- [ ] Install and configure Suricata and Zeek
- [ ] Stage the attack
- [ ] Detect and triage alerts
- [ ] Investigate: timeline, root cause, impact
- [ ] Map to MITRE ATT&CK
- [ ] Write the incident report

---

## Documentation

| Stage | Document |
|---|---|
| 1. Lab setup | [docs/01-lab-setup.md](docs/01-lab-setup.md) |
| 2. Network design | *coming soon* |
| 3. Attack | *coming soon* |
| 4. Detection | *coming soon* |
| 5. Investigation | *coming soon* |
| 6. ATT&CK mapping | *coming soon* |
| Incident report | *coming soon* |

---

## Repository Structure

```
home-dfir-lab/
├── README.md
├── docs/                 # Write-ups for each stage
├── evidence/
│   └── screenshots/      # Command output and tool screenshots
├── configs/              # Suricata rules and Zeek scripts (added later)
└── report/               # Final incident report (added later)
```

---

## Tools

VirtualBox · Kali Linux · Metasploitable 2 · Suricata · Zeek · Wireshark · MITRE ATT&CK

---

## Scope & Ethics

All activity is performed in an isolated VirtualBox lab against intentionally vulnerable virtual machines that I own and control. No external systems or networks are targeted.

---

## Lessons Learned

*To be completed at the end of the project.*

---

## Author

**Umar Rahman** · Final-year Computer Science student, University of Southampton