# 03 - Detection

Acting as the defender: Monitoring lab network for the attack and raising alerts on it. This report covers how Suricata works, what it caught, what it missed and writing a custom rule to close the gap 

## How Suricata works

Suricata is an IDS (Intrusion Detection System): it watches network traffic, matches it against a set of rules, and raises an alert when something matches a bad pattern.

Two things worth being precise about:

- IDS vs IPS. An IDS watches a copy of the traffic, whereas an IPS sits between the traffic source and destination. Hence an IDS is passive, it can only raise alerts, whilst an IPS can block traffic. I chose Suricata as an IDS so I could study what the attack looks like rather than interfere with it.
- A sensor only sees traffic that crosses its interface. I ran Suricata on Kali's eth1 (the labnet interface), so it can only see the traffic to and from kali's eth1. Because Kali is the only attacker within this setup, this is sufficient, but a more realistic setup would be to use a dedicated sensor VM that can listen to the traffic on the entire network, so attacks other than the one from the Kali VM could be tracked.

## Detecting the Nmap scan

I began by updating and running Suricata on the Kali eth1 interface:
 ```
sudo suricata-update          # pull the ET Open ruleset (~53,000 rules)
sudo suricata -i eth1         # run the IDS on the lab interface
sudo tail -f /var/log/suricata/fast.log   # watch alerts live
```

With Suricata running, I re-ran the Nmap scan on metasploitable:

```
sudo nmap -sV 10.10.10.20
```
Suricata caught it immediately, as it matched the prebuilt ruleset located in suricata.rules:

ET SCAN Possible Nmap User-Agent Observed [Priority: 1] {TCP} 10.10.10.10 -> 10.10.10.20:80

## Detection gap: the exploit slipped past

I then ran the vsftpd backdoor exploit with Suricata still watching. The result was the important finding of this stage: no alert fired for the exploitation. The nmap scan was detected, but the actual break-in was not.

Why: signature-based detection only catches what it has a rule for. The built in ruleset has thousands of rules, but none that match the vsftpd 2.3.4 :) backdoor trigger. To Suricata the packet that contains `USER hacker:)` just looks like normal traffic so it passed through with no alert.

This shows a fundamental weakness of signature-based IDS: it's blind to anything it doesn't have a rule for. Known attacks are caught, but any new exploits walk past.

In the next part we write a custom rule for the vsftpd 2.3.4 vulnerability

## Writing a custom rule

I wrote my own rule to detect the backdoor trigger, and kept it in a separate local.rules file so it's not overwritten when the official ruleset updates:

```
alert tcp any any -> any 21 (msg:"LOCAL rule: vsftpd 2.3.4 backdoor trigger detected"; content:"USER"; content:":)"; sid:1000001; rev:1; classtype:trojan-activity;)
```

Rule anatomy:

- alert tcp any any -> any 21: alert on TCP traffic from anywhere to any host on port 21 (FTP).
- msg: the text shown when it fires.
- content:"USER"; content:":)": the detection logic: match traffic containing both USER and the :) smiley. That combination is the backdoor's fingerprint.
- sid:1000001: the signature ID. The 1,000,000+ range is reserved for local rules so it never clashes with official ones.
- classtype:trojan-activity: the category.

Loaded it alongside the main ruleset:

```
sudo suricata -i eth1 -s local.rules
```

Re-running the trigger made my rule fire:

```
[1:1000001:1] LOCAL rule: vsftpd 2.3.4 backdoor trigger detected [Classification: A Network Trojan was detected] [Priority: 1] {TCP} 10.10.10.10 -> 10.10.10.20:21
```

My own rule (SID 1000001) caught the exact attack the 53,000 built-in rules let through.

**Limitations of the rule.** This rule works, but it is a simple one I wrote for the lab, not something you would run in production. It has two main weaknesses:

- **It doesn't check that `USER` and `:)` belong together.** The rule just looks for both `USER` and `:)` somewhere in the same packet. It doesn't confirm the `:)` is actually part of the FTP username, so a normal login like `USER bob` that happened to have a `:)` elsewhere in the packet could set it off by mistake (a false alarm). A better rule would make sure the `:)` is inside the username itself.
- **It only watches port 21.** FTP normally runs on port 21, but it can run on other ports too (this victim also runs a second FTP service on 2121). The rule would miss the same attack there.

A more robust version would use Suricata's built-in understanding of FTP, so it reads the actual FTP commands instead of just scanning the raw packet for text. That would fix both problems at once, and it is the improvement I would make next.
The production approach to both problems is to use Suricata's app-layer FTP keywords, which inspect the parsed FTP command field rather than raw packet bytes. That is port-agnostic and far less prone to false positives, and it is the planned v2. The gap between this lab rule and that version is a good illustration of the difference between a signature that works in a test and one you would actually deploy.

## Key takeaways

- Signature-based IDS catches known attack patterns but is blind to anything without a matching rule. I saw both sides of this live: the scan was caught, the exploitation was not.
- An IDS only inspects live traffic on the interface it watches. It must be running at the time, and sensor placement decides what it sees.
- Writing custom detection logic closes the gaps the default ruleset leaves. Noticing the missed exploit, writing a signature for it, and proving it fires is the core of detection engineering.

## Scope and ethics

All activity was performed in an isolated VirtualBox lab against an intentionally vulnerable virtual machine that I own and control. No external systems or networks were involved.
