# Incident Report: vsftpd 2.3.4 Backdoor Exploitation

**Author:** Umar Rahman
**Environment:** Isolated home DFIR lab (VirtualBox)
**Date of exercise:** November 2026

---

## 1. Summary

An attacker performed reconnaissance against a target host, identified an outdated FTP service, and exploited a known backdoor in that service to gain full root control of the host. The entire chain, from scan to root access, was carried out and then investigated from the defender's side.

Network monitoring (Suricata) detected the reconnaissance scan but did not detect the exploitation itself, because no default signature existed for this backdoor. A custom detection rule was written to close that gap, and it successfully detected the attack on re-test. The intrusion was then reconstructed from packet captures and network logs.

Root cause: the host ran outdated software (vsftpd 2.3.4) containing a publicly documented backdoor, with no patching and unnecessary services exposed.

Impact: complete compromise of the host at root level, giving the attacker total control.

---

## 2. Scope and Environment

This was a controlled exercise in an isolated lab that I built, playing the attacker, the defender, and the investigator in turn.

| Component | Detail |
|---|---|
| Attacker | Kali Linux (10.10.10.10) |
| Victim | Metasploitable 2 (10.10.10.20) |
| Network | Isolated VirtualBox Internal Network (`labnet`, 10.10.10.0/24), no internet or host access |
| Monitoring | Suricata (IDS) and Zeek (network logging) on the lab interface; tcpdump for packet capture; Wireshark for analysis |

All times below use the lab host's local clock.

---

## 3. Timeline of Events

| Time | Phase | Event | Evidence |
|---|---|---|---|
| (recon run) | Reconnaissance | Nmap `-sV` service scan of the victim; identified vsftpd 2.3.4 on port 21 | Suricata `ET SCAN Possible Nmap User-Agent` alert |
| 2026-11-07 08:43:43 | Initial Access | FTP connection to port 21; backdoor trigger `USER hacker:)` sent | Custom Suricata rule `SID 1000001`; Zeek `ftp.log` (`user: hacker:)`); packet capture |
| 2026-11-07 08:44:14 | Execution | Connection to the backdoor shell on port 6200; `whoami` executed, returned `root` | Zeek `conn.log` (unidentified service on 6200); Wireshark TCP stream (`whoami` → `root`) |

The attacker moved from the backdoor trigger to a confirmed root shell in roughly 30 seconds.

---

## 4. Technical Findings

### 4.1 Reconnaissance

An Nmap service-version scan (`nmap -sV`) was run against the victim. It enumerated a large attack surface of outdated services, including vsftpd 2.3.4 on port 21. The scan was "loud" active reconnaissance and was detected by Suricata's default ruleset (`ET SCAN Possible Nmap User-Agent Observed`), as the version probes carried Nmap's signature.

### 4.2 Initial Access

The vsftpd 2.3.4 service contains a backdoor (**CVE-2011-2523**): a login username containing the characters `:)` causes the service to open a root shell on port 6200. The attacker exploited this by connecting to port 21 and sending `USER hacker:)`.

- Suricata's default ruleset did not detect this (no matching signature).
- A **custom rule (SID 1000001)** was written to detect the trigger and successfully alerted on it (`LOCAL rule: vsftpd 2.3.4 backdoor trigger detected`).
- Zeek's `ftp.log` captured the malicious username (`user: hacker:)`) as a structured field.

### 4.3 Execution

The backdoor opened a root shell listening on port 6200. The attacker connected to it and ran `whoami`, which returned `root`, confirming full root access.

- Zeek's `conn.log` recorded this as a connection to an unidentified service on an unusual high port (6200), itself a suspicious indicator.
- Because the shell traffic is unencrypted, the Wireshark TCP stream showed the command and response in plaintext (`whoami` → `root`).

---

## 5. Root Cause

The compromise was possible because of outdated and unpatched software:

- The host ran vsftpd 2.3.4, an outdated version containing a publicly documented backdoor with a ready-made exploit.
- No patch management was in place; the vulnerable service was left running.
- Numerous unnecessary services were exposed, enlarging the attack surface.

Known vulnerabilities in outdated software are trivially exploitable because working exploits are published publicly, so even a low-skill attacker can use them.

---

## 6. Impact

The attacker achieved root access to the host. In practice this means total control:

- Read any file on the system, including `/etc/shadow` (all account password hashes), which could be cracked offline.
- Modify or delete any file; install software; create backdoor accounts.
- Establish persistence and clear logs to hide the intrusion.

In a real environment this host could then be used as a foothold to attack other systems on the network (lateral movement).

---

## 7. Detection Summary

| Stage | Detected by default ruleset? | Detected overall? |
|---|---|---|
| Reconnaissance scan | Yes (`ET SCAN`) | Yes |
| Backdoor exploitation | **No** | Yes, via custom rule `SID 1000001` |

The key defensive finding is that signature-based detection caught the noisy scan but missed the actual break-in, because it had no rule for this backdoor. Writing a custom signature resolved this. Zeek provided the structured timeline and surfaced the suspicious unidentified service on port 6200; Wireshark provided packet-level proof.

---

## 8. MITRE ATT&CK Mapping

| Tactic | Technique | In this incident |
|---|---|---|
| Discovery | T1046 Network Service Discovery | Nmap service/version scan |
| Lateral Movement | T1210 Exploitation of Remote Services | Exploiting the vsftpd 2.3.4 backdoor to gain access |
| Execution | T1059 Command and Scripting Interpreter | Commands run in the root shell |

---

## 9. Remediation and Recommendations

1. **Patch or remove the vulnerable software.** vsftpd 2.3.4 is outdated and should be updated to a current version or removed.
2. **Implement patch management.** Keep software current so publicly known exploits are closed off; this is one of the highest-impact controls.
3. **Reduce the attack surface.** Disable or remove unnecessary services; only run what is needed.
4. **Restrict and segment network access** so services are not reachable from untrusted sources.
5. **Maintain network monitoring**, and write custom detection rules for gaps the default ruleset leaves, as demonstrated here.

---

## 10. Limitations and Confidence

In the interest of an honest assessment:

- **What the evidence confirms:** the reconnaissance, the backdoor trigger, and root-level access were all directly observed in the alerts, logs, and packet captures.
- **What it does not confirm:** no further actions (data exfiltration, lateral movement, persistence) were performed in this exercise, so there is no evidence of them; the investigation stops at confirmed root access.
- **Lab scale:** this is a small, single-host lab. Zeek's and the IDS's real strengths appear at production scale (searchable logs across high-volume traffic), which a lab this size does not exercise. Here, Wireshark alone was sufficient to analyse the traffic; Zeek was used to demonstrate the network-security-monitoring approach.
- **Sensor placement:** detection was run on the attacker host for simplicity. A more realistic deployment would use a dedicated sensor with traffic mirroring, separating the monitor from the attacker.
- **Multiple sessions:** the exercise was conducted over several sessions, so some evidence (for example the reconnaissance alert) comes from a separate capture to the exploitation.

---

## 11. Evidence (Appendix)

- Suricata alerts (`fast.log`): `ET SCAN` on the scan; custom rule `SID 1000001` on the backdoor trigger.
- Custom Suricata rule: `configs/suricata/local.rules`.
- Packet capture: `incident.pcap`.
- Zeek logs: `conn.log` (connection timeline), `ftp.log` (captured `user: hacker:)`).
- Wireshark TCP streams: the `USER hacker:)` trigger on port 21; `whoami` / `root` on port 6200.
