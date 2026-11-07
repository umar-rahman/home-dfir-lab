# Evidence

Screenshots captured during the lab, grouped by stage. Each stage folder maps to its write-up in [`docs/`](../docs), and together they back the findings in the [incident report](../report/incident-report.md).

## 01 - Lab setup

| Screenshot | What it shows |
|---|---|
| [`01-01-virtualbox-manager.png`](screenshots/01-lab-setup/01-01-virtualbox-manager.png) | Both VMs (Kali and Metasploitable 2) in VirtualBox. |
| [`01-02-kali-ip-addr.png`](screenshots/01-lab-setup/01-02-kali-ip-addr.png) | Kali is dual-homed: `eth0` on NAT, `eth1` = `10.10.10.10` on the lab network. |
| [`01-03-metasploitable-ip.png`](screenshots/01-lab-setup/01-03-metasploitable-ip.png) | The victim on the lab network at `10.10.10.20`. |
| [`01-04-isolation-ping-trio.png`](screenshots/01-lab-setup/01-04-isolation-ping-trio.png) | Isolation proof: the victim cannot reach the internet, can reach Kali, and the host cannot reach the victim. |

## 02 - Recon and exploitation

| Screenshot | What it shows |
|---|---|
| [`02-01-nmap-host-discovery.png`](screenshots/02-recon-exploitation/02-01-nmap-host-discovery.png) | Host discovery confirms the victim is up. |
| [`02-02-nmap-version-scan.png`](screenshots/02-recon-exploitation/02-02-nmap-version-scan.png) | Version scan identifies **vsftpd 2.3.4** on port 21 (CVE-2011-2523). |
| [`02-03-netcat-trigger.png`](screenshots/02-recon-exploitation/02-03-netcat-trigger.png) | Backdoor trigger: `USER hacker:)` sent over FTP; the control session hangs. |
| [`02-04-netcat-root-shell.png`](screenshots/02-recon-exploitation/02-04-netcat-root-shell.png) | Root shell on port 6200; `whoami` / `id` returns `root`. |

## 03 - Detection

| Screenshot | What it shows |
|---|---|
| [`03-01-suricata-update.png`](screenshots/03-detection/03-01-suricata-update.png) | The ET Open ruleset loaded (~53,000 rules). |
| [`03-02-suricata-scan-alert.png`](screenshots/03-detection/03-02-suricata-scan-alert.png) | Suricata's default ruleset catches the Nmap scan (`ET SCAN`). |
| [`03-03-local-rule.png`](screenshots/03-detection/03-03-local-rule.png) | The custom rule (SID 1000001) targeting the backdoor trigger. |
| [`03-04-custom-rule-alert.png`](screenshots/03-detection/03-04-custom-rule-alert.png) | The custom rule firing on the exploit the default ruleset missed. |

## 04 - Investigation

| Screenshot | What it shows |
|---|---|
| [`04-01-zeek-conn-log.png`](screenshots/04-investigation/04-01-zeek-conn-log.png) | Zeek `conn.log`: the unidentified service on port 6200. |
| [`04-02-zeek-ftp-log.png`](screenshots/04-investigation/04-02-zeek-ftp-log.png) | Zeek `ftp.log`: the malicious username `user: hacker:)` captured as a structured field. |
| [`04-03-wireshark-ftp-trigger.png`](screenshots/04-investigation/04-03-wireshark-ftp-trigger.png) | Wireshark TCP stream: the `USER hacker:)` trigger on port 21. |
| [`04-04-wireshark-root-shell.png`](screenshots/04-investigation/04-04-wireshark-root-shell.png) | Wireshark TCP stream: `whoami` → `root` on port 6200. |
