# 🛡️ Home SOC Lab — Part 1: Building the Foundation

**by BetterCallS4V**


## The Story

So I decided I wanted to break into cybersecurity. Not literally. Well... actually, yes, literally — but in a lab environment.

Like every person who goes down this rabbit hole, I started watching YouTube videos, reading about SOC analyst roles, and somewhere between the third video and the fourth cup of tea I thought — *I should just build this thing myself.*

So I did. Here's how it went.

---

## The Shopping List

I'm based in Ireland, and I didn't want to spend a fortune. Here's what I picked up:

| Item | Spec | Cost |
|---|---|---|
| HP EliteDesk 800 G6 SFF (refurbished) | i5, 16GB RAM, 256GB SSD | €180 |
| Lenovo ThinkPad T14 (refurbished) | i5 10th gen, 16GB RAM, 512GB SSD | €98|
| MacBook M3 Pro | — | already owned |
| TP-Link Router | basic home router | €18 |
| Ethernet cable | because WiFi wasn't going to cut it | a few euro |
| USB drive | borrowed from a friend | free (thanks mate) |

> Nobody uses USB drives anymore apparently. Had to borrow one from a friend like it was 2009. No judgment.

The HP EliteDesk is the server. The ThinkPad becomes the Kali attacker machine. The MacBook is the management console .
---

## First Problem (Before Anything Even Started)

Plugged the HP in. Grabbed an HDMI cable. Looked at the back of the machine.

No HDMI port.

The HP EliteDesk 800 G6 SFF uses **DisplayPort**, not HDMI. Had to go get a DisplayPort cable before I could even see what I was doing. Small thing, but worth knowing if you pick up the same machine.

---

## Lab Architecture

```
HP EliteDesk 800 G6 (Proxmox Host)
└── Proxmox VE 9.1.1
    ├── VM 100 — Windows Server 2022  (Domain Controller)
    ├── VM 101 — Ubuntu Server 24.04  (Wazuh SIEM)
    └── VM 102 — Windows 10           (Endpoint) [Part 2]

Lenovo ThinkPad T14
└── Kali Linux  (Attacker) [Part 2]
```

### Network

```
Modem
  |
TP-Link Router (192.168.0.x)
  /         |          \
HP PC     ThinkPad    MacBook
(eth)      (WiFi)      (WiFi)
.50         TBD         TBD
```

### IP Reference

| Machine | Role | IP |
|---|---|---|
| Proxmox host | Hypervisor | 192.168.X.50 |
| Windows Server 2022 | Domain Controller | 192.168.X.104 |
| Ubuntu Server 24.04 | Wazuh SIEM | 192.168.X.105 |

---

## Phase 1 — Installing Proxmox

### What even is Proxmox?

Before getting into the steps — why Proxmox at all?

The goal of this lab is to run multiple machines simultaneously: a Domain Controller, a SIEM, a Windows endpoint, and eventually more. Buying separate physical hardware for each would cost a lot. Proxmox lets you run all of them as **virtual machines** on a single physical box.

Think of the HP desktop as a building. Proxmox is the landlord. Each VM is a tenant with their own room, their own OS, completely independent. One physical machine, multiple virtual ones running at the same time.

### Tools Needed

- A USB drive (8GB or more)
- **Rufus** — https://rufus.ie
- **Proxmox VE ISO** — https://www.proxmox.com/en/downloads

### Steps

**1. Flash the USB**

Downloaded Rufus and the Proxmox ISO. Opened Rufus, selected the USB, selected the ISO, and chose **Write in DD Image mode** when prompted. This part matters — using ISO mode can result in a USB that doesn't boot properly on some machines.

Took about 5 minutes.

**2. Boot from USB**

Plugged the USB into the HP. Powered on. Tapped **F9** repeatedly to get the one-time boot menu. Selected the USB. The Proxmox installer loaded.

**3. Network configuration screen**

Most of the installer is straightforward. The screen that actually matters is the **network configuration screen**.


| Field | Value used |
|---|---|
| Hostname (FQDN) | `pve.lab.local` |
| IP Address | `192.168.X.50/24` |
| Gateway | `192.168.X.1` |
| DNS Server | `8.8.8.8` |

> Setting the wrong IP here caused a full reinstall. More on that in Phase 2.

**4. Install, reboot, access**

Installation takes about 10 minutes. After reboot, the HP screen shows:

```
Welcome to Proxmox Virtual Environment.
Please use your web browser to configure this server — connect to:
https://192.168.X.50:8006/
```

Opened that URL on the MacBook. Got a security warning about the certificate — that's normal for a self-signed cert, just click through it. Logged in with `root` and the password set during install.

---

## Phase 2 — The Network Situation

This is the part of the build that took the longest. Not because it's complicated in theory, but because a few things had to line up correctly before Proxmox was actually reachable from another device.

### Problem 1 — Proxmox doesn't do WiFi

First attempt had the HP connected wirelessly. Proxmox's management interface **does not support WiFi** — it needs a wired ethernet connection. Once a cable was plugged in and the HP was connected to the router properly, things started moving.

### Problem 2 — Wrong IP range

The Proxmox installer auto-detected the wrong network. It set the IP to `192.168.100.2` but the home network was actually `192.168.X.x`. Devices on different subnets cannot see each other, so every ping returned 100% packet loss.

Diagnosed with:
```bash
ping 192.168.XXX.2    # timeout
arp -a                # confirmed MacBook was on 192.168.X.x
```

Fixed by logging into Proxmox directly and editing the network config:

```bash
nano /etc/network/interfaces
```

Changed:
```
address 192.168.X.2/24
gateway 192.168.X.1
```
To:
```
address 192.168.X.50/24
gateway 192.168.X.1
```

Saved, rebooted. Finally reachable.

### Problem 3 — Multiple routers, multiple subnets

The house already had a router but it was shared with other people. Bought a separate TP-Link for €18 and connected the HP via ethernet and the laptops via WiFi. Everything landed on the same `192.168.X.x` network and could communicate properly.

### The fix that actually worked

```
Modem → TP-Link Router → HP (ethernet) + laptops (WiFi)
```

All on the same subnet. Done.

### Lesson learned

Before installing Proxmox, run `ipconfig` (Windows) or `ifconfig` (Mac/Linux) on any device already on your network. Check your **gateway IP**. That tells you the subnet you're working with. Set Proxmox's IP to match from the start.

Would have saved about two hours.

---

## Phase 3 — Windows Server 2022 and Active Directory

### Creating the VM

With Proxmox running, the first VM to build was a Windows Server 2022 instance that would become the **Domain Controller** for the lab.

Downloaded the Windows Server 2022 evaluation ISO from Microsoft's eval centre — free, 180-day licence, more than enough for a lab. Uploaded it to Proxmox under local storage > ISO Images.

Created the VM:

| Setting | Value |
|---|---|
| Name | Windows-server |
| ISO | SERVER_EVAL_x64FRE_en-us.iso |
| OS Type | Microsoft Windows 2022 |
| BIOS | OVMF (UEFI) |
| CPU | 2 cores |
| RAM | 4096 MB |
| Disk | 50GB |

> When the VM first boots, there's a "Press any key to boot from CD or DVD..." prompt that disappears in about 2 seconds. You have to click inside the Proxmox console window and press a key fast. Miss it and the VM boots to a black screen and sits there doing nothing. Ask me how I know.

### Installing Windows Server

Selected **Windows Server 2022 Standard Evaluation (Desktop Experience)** — the version with a full GUI. Much easier to manage in a lab than the command-line only version.

Selected **Custom install**, let it partition the 50GB disk automatically, and left it running. Takes about 20 minutes.


### Promoting to Domain Controller

Once Windows Server was up, the next step was turning it into an **Active Directory Domain Controller**. This is what manages users, computers, authentication and group policies — the same infrastructure found in most corporate networks.

In Server Manager, went to **Add Roles and Features** and selected **Active Directory Domain Services**. After installation completed, clicked **"Promote this server to a domain controller"**.

Configured a new forest:

| Setting | Value |
|---|---|
| Operation | Add a new forest |
| Root domain name | `lab.local` |
| Forest functional level | Windows Server 2016 |
| NetBIOS name | LAB |

Server rebooted and came back showing **LAB\Administrator** at the login screen. Domain Controller live.

### Creating Domain Users

Four standard user accounts were created in **Active Directory Users and Computers** to simulate a realistic corporate environment.

| Name | Username |
|---|---|
| John Smith | jsmith |
| Sarah Jones | sjones |
| Bob Wilson | bwilson |
| Lisa Turner | lturner |

All accounts set with password never expires and no forced change on first login.

> These are the people who are about to have a very bad time when Kali comes online in Part 2.

---

## Phase 4 — Ubuntu Server

### Creating the VM

The second VM is Ubuntu Server 24.04 LTS, which will host **Wazuh** — the open source SIEM that collects and analyses security logs from all the Windows machines in the lab.

Same process as before — uploaded the Ubuntu ISO to Proxmox and created the VM:

| Setting | Value |
|---|---|
| Name | wazuh-siem |
| ISO | ubuntu-24.04.4-live-server-amd64.iso |
| OS Type | Linux 6.x kernel |
| CPU | 2 cores |
| RAM | 4096 MB |
| Disk | 50GB |

### Installing Ubuntu Server

The Ubuntu installer is text-based. Key decisions during install:

- Selected **Ubuntu Server** (standard, not minimised)
- Used default guided storage across the full 50GB disk
- **Enabled OpenSSH server** — allows SSH access from the terminal, much more practical than using the Proxmox console every time


After reboot:

```
Ubuntu 24.04.4 LTS
System load:   0.0
Memory usage:  5%
IPv4 address:  192.168.X.105
```

Wazuh installation is the next step and will be covered in Part 2 once internet access is configured on the VM.

---

## Where Part 1 Ends

| Component | Status |
|---|---|
| Proxmox VE 9.1.1 | ✅ Running |
| Windows Server 2022 | ✅ Running |
| Active Directory (lab.local) | ✅ Configured |
| Domain users x4 | ✅ Created |
| Ubuntu Server 24.04 | ✅ Running |
| Wazuh SIEM | ⏳ Part 2 |
| Windows 10 endpoint | ⏳ Part 2 |
| Kali Linux setup | ⏳ Part 2 |
| Attack simulations | ⏳ Part 3 |

---

## Tools Used in Part 1

| Tool | Purpose |
|---|---|
| Proxmox VE | Hypervisor — runs all VMs on the HP desktop |
| Rufus | Flashes ISO files to USB drives |
| Windows Server 2022 | Domain Controller operating system |
| Active Directory | User and computer management |
| Ubuntu Server 24.04 | Wazuh SIEM host OS |
| OpenSSH | Remote terminal access to Ubuntu VM |

---

*Part 2 coming soon — Wazuh installation, Windows 10 endpoint, Kali setup and the first attacks.*

**— BetterCallS4V**
