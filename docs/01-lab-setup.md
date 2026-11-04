# 01 - Lab Setup

Building the isolated environment: a Kali Linux attacker and a deliberately vulnerable Metasploitable 2 victim, on a sealed virtual network. This covers each setup choice and my reasoning behind it.

## Host environment

| Component | Detail |
|---|---|
| Host OS | Windows |
| CPU | AMD Ryzen 7 7700 |
| RAM | 32 GB |
| Hypervisor | Oracle VirtualBox (Type 2) |

## Why VirtualBox

VirtualBox is a Type 2 hypervisor: it runs as an application on top of Windows, rather than directly on the hardware, unlike a type 1 hypervisor. It shares the host's CPU, RAM and disk between virtual machines by presenting each VM with virtual hardware it believes is its own.

Why I'm using it for this security lab:

- **Isolation.** A guest OS only sees the virtual hardware it is given. This way any vulnerabilities within the vm stays sealed in its sandbox, and any compromise can't spread to my real machine or network.
- **Snapshots.** A VM's disk is just a file on the host. A snapshot freezes that file and writes later changes to a separate difference file, so restoring is near-instant. This means I can detonate an attack and easily roll back to a clean slate.

## The attacker: Kali Linux

**Why Kali:** it's the industry-standard Linux distro built for security testing. It's pre-loaded with the tools I need (Nmap, Metasploit, netcat, Wireshark), so there is nothing to install for the attack stages.

**VM settings:** 4 vCPUs, 4 GB RAM. Enough to run tools smoothly while leaving plenty for the host pc and the VM.

**Setup steps:**
1. Imported the pre-built Kali VirtualBox image (not an ISO install, the machine came ready-made).
2. Updated with `sudo apt update && sudo apt full-upgrade -y`. Kali is a rolling release, so a fresh image is already behind and the first update is large.
3. Took a clean baseline snapshot after updating.

## The victim: Metasploitable 2

**Why Metasploitable 2:** it is an intentionally vulnerable Linux VM (Ubuntu 8.04) with many known-vulnerable services. I chose it over Metasploitable 3 because this project is network-focused, and Metasploitable 2 is far quicker to set up and well documented. Metasploitable 3 (Windows) is planned for future use.

**Setup steps:**
1. Downloaded and extracted the Metasploitable 2 image to get its virtual disk (`.vmdk`).
2. Created a VM of type Linux / Ubuntu (32-bit), because it is an old 32-bit OS. The wrong type can stop it booting.
3. Newer VirtualBox no longer attaches an existing disk in the creation wizard, so I let it create an empty disk, then in Settings → Storage removed that and attached the `.vmdk` instead.
4. Disabled the USB controller and left shared folders empty, to keep a deliberately vulnerable machine away from the host's real hardware and files. Same isolation principle as the network.

## Network design

This is the part that required most care. The victim is full of remotely exploitable services, so I ensured that I designed the network in a way that it can never reach, or be reachable from, any real network.

A table of the VirtualBox network modes, viewed from a security perspective

| Mode | Internet | Other VMs | Host to VM | Safe for a vulnerable box? |
|---|---|---|---|---|
| NAT | Out only | No | No (unless port-forwarded) | Risky, still has outbound internet |
| NAT Network | Out only | Yes | No | No |
| Bridged | Full | Yes | Yes | Never, sits on the real home LAN |
| Host-only | No | Yes | Yes | Usable, host is exposed |
| **Internal Network** | **No** | **Yes** | **No** | **Best, sealed to lab VMs only** |

**The design:**

```
                ┌──────────────────── VirtualBox ────────────────────┐
 Internet ◄─NAT─┤ eth0  Kali (10.10.10.10 on eth1)                    │
                │ eth1  Kali ──┐                                      │
                │              │  Internal Network "labnet"           │
                │ Metasploitable ── (10.10.10.20)                     │
                │              no internet, no host, no home LAN      │
                └─────────────────────────────────────────────────────┘
```

- **Kali is dual-homed:** Adapter 1 on NAT (so it can fetch updates and tools from the internet) and Adapter 2 on Internal Network `labnet` (the sealed wire to the victim). Kali is allowed internet because it is the hardened attacker, not the target that contains the vulnerabilities.
- **Metasploitable has one adapter:** Internal Network `labnet` only. No internet, no host, no home LAN.

The asymmetry is deliberate: the attacker can still update itself, while the victim and all attack traffic stays isolated.

## IP addressing

An Internal Network has no DHCP server, so nothing hands out addresses automatically. I set static IPs manually, inside each OS (VirtualBox only decides which network a card is plugged into, not its address):

- **Metasploitable:** edited `/etc/network/interfaces` to set `eth0` to `10.10.10.20` / `255.255.255.0`. Deliberately no gateway line: with no default route it cannot send traffic outside the lab subnet, a second layer of isolation on top of the Internal Network.
- **Kali:** added a persistent NetworkManager connection on `eth1`:
  ```
  sudo nmcli con add type ethernet ifname eth1 con-name labnet \
    ipv4.method manual ipv4.addresses 10.10.10.10/24 \
    ipv4.never-default yes ipv6.method disabled
  ```
  `ipv4.never-default yes` stops `eth1` becoming a default route, so internet traffic always leaves via the NAT adapter and never tries the dead-end lab network.

I used the `10.10.10.0/24` subnet (a private range) kept deliberately separate from the home network (`192.168.x.x`) and VirtualBox's own NAT network (`10.0.2.x`), so none of the three overlap.

## Verifying isolation

To confirm the network actually complies with my intended design, I tested using pings

| Test (from) | Command | Result | Proves |
|---|---|---|---|
| Victim → internet | `ping -c3 8.8.8.8` | Fail | Sealed from the internet |
| Kali → Victim | `ping -c3 10.10.10.20` | Success | Lab network works |
| Host → victim | `ping 10.10.10.20` (Windows) | Fail | Host cannot reach the victim |

Together these prove the victim can talk to the lab and nothing else. The second test matters as much as the first: it shows the internet failure is isolation, not a broken setup.

## Snapshots

Two restore points taken:
- **Clean baseline** (fresh, updated Kali).
- **Lab network working** (both VMs configured and verified).

Because the victim is intentionally vulnerable and never patched, the recovery strategy for it is not to fix it but to restore a snapshot back to a clean vulnerable state.

## Key takeaways

- A Type 2 hypervisor makes a safe, repeatable attack lab possible on a single machine.
- Isolation is a design choice at the network layer. "No route out at all" (Internal Network) is stronger than "a route that is currently firewalled."
- A sensor, or any machine, only sees traffic that crosses its own interface. This matters for the detection stage.
- Static addressing, routing and snapshots are the infrastructure skills underneath the security work.
