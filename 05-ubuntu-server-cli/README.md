# Lab 05 – Ubuntu Server Install & First SSH Connection

## Overview

This lab documents installing *Ubuntu Server 26.04 LTS* as a headless (no GUI) virtual machine in VirtualBox, and getting comfortable navigating it entirely through the command line. Unlike Kali Linux, Ubuntu Server represents what's actually running in most real-world IT environments - file servers, web servers, and infrastructure are rarely running a desktop environment.

This was my first time ever installing a server operating system. I ran into a real networking problem along the way (SSH wouldn't connect) and had to diagnose and fix it myself - that troubleshooting process ended up being the most valuable part of the lab.


# Objectives

- Install Ubuntu Server in VirtualBox using the unattended installation wizard
- Successfully log into a headless Linux server for the first time
- Diagnose and fix a real SSH connectivity issue
- Connect remotely via SSH instead of relying on the VirtualBox console window
- Get comfortable with basic verification commands ('whoami', 'ip a', 'systemctl')


# Environment

| Item | Details |
|------|---------|
| Host Machine: Lenovo ThinkPad P14s (Ryzen 7 Pro, 32GB RAM) 
| Hypervisor: Oracle VirtualBox 
| Guest OS: Ubuntu Server 26.04 LTS 
| VM Name: 'ubuntu-server' (hostname set to 'ubuntu-server-01') 
| Username: 'labadmin' 
| RAM Allocated: 2048 MB 
| CPUs: 2 
| Disk: 20 GB (VDI, dynamically allocated) 


# Step 1 – Creating the VM

I used VirtualBox's newer *unattended installation* wizard, which lets you set the username, password, and hostname directly in the VM creation screen instead of typing them into the installer manually.

- Selected the downloaded ISO (`ubuntu-26.04-live-server-amd64.iso`)
- VirtualBox auto-detected it as Ubuntu 64-bit and pre-filled OS type
- Left **"Proceed with Unattended Installation"** checked
- Set hostname to `ubuntu-server-01`, username to `labadmin`

[VM creation wizard](./screenshots/01-vm-creation-wizard.png)

For hardware, I went with conservative numbers since I knew I'd eventually be running multiple VMs side by side (Kali + this + possibly Windows Server later) and didn't want to over-allocate one machine:

![Virtual hardware settings — 2048MB RAM, 2 CPUs](./screenshots/02-virtual-hardware-settings.png)

Disk was set to 20GB, VDI format, dynamically allocated. Once created, the VM's Details pane confirmed everything matched what I'd set:

![VM details confirming settings](./screenshots/03-vm-details-confirmed.png)

> *Note:* VirtualBox automatically deletes the temporary unattended-install configuration files from the VM folder once setup completes — it'll prompt you to confirm. That's expected and safe to allow.


# Step 2 – Boot & Install

First boot lands on GRUB, which automatically proceeds into the installer after a few seconds:

![GRUB bootloader](./screenshots/04-grub-bootloader.png)

Since I used the unattended install, there was no manual click-through — Ubuntu's installer (called **Subiquity**) ran the whole setup automatically based on the answers I gave in the wizard. The terminal just scrolls through a long list of `start:`/`finish:` lines as it configures the filesystem, network, and packages:

![Subiquity install progress](./screenshots/05-install-progress-subiquity.png)

Partway through, it switches to *curtin*, which handles the actual system installation (partitioning, bootloader, copying files):

![Curtin install steps](./screenshots/06-install-progress-curtin.png)

After install, the VM reboots into the new system and you can watch standard Linux boot messages (`systemd` starting services) — this is the same boot sequence you'd see on any real Ubuntu server:

![First boot systemd services starting](./screenshots/07-first-boot-systemd.png)



#  Step 3 – First Login

After boot, I was greeted with a login prompt. Logged in with the 'labadmin' username and password set during the wizard:

![First successful login](./screenshots/08-first-successful-login.png)

This was a genuine milestone — first time logging into a server with zero graphical interface, just a '$' prompt waiting for commands.

The login banner also displayed useful system info automatically (disk usage, memory, IP address) — handy for a quick health check every time you log in.


# Step 4 – Finding the VM's IP Address

```bash
'ip a'
```

This showed two relevant entries: 'lo' (loopback, '127.0.0.1' — not useful for remote access) and 'enp0s3' (the actual network adapter) with an address of '10.0.2.15':

![ip a showing NAT address](./screenshots/09-ip-a-nat-address.png)

'10.0.2.x' is VirtualBox's default **NAT** range — it told me the VM could reach the internet, but was sitting behind VirtualBox's internal router, isolated from my actual home network.


#  Step 5 – The SSH Troubleshooting Story

This is the part I'm most glad I documented, because it's a real diagnose-and-fix process, not just following steps.

### Problem 1: Connection Timed Out

From PowerShell on my host machine, I tried:
```bash
ssh labadmin@10.0.2.15

Result: **connection timed out.**

**Diagnosis:** the VM's network adapter was set to NAT, which keeps the VM in a private network that VirtualBox manages, my host machine couldn't route directly to '10.0.2.15' because that address only exists inside VirtualBox's internal NAT network.

*Attempted fix #1 — Port Forwarding:* I looked at adding a port forwarding rule (Settings -> Network -> advanced -> Port Forwarding) to forward host port `2222` to guest port '22':

![Port forwarding rule configuration](./screenshots/10-port-forwarding-attempt.png)

*Actual fix — Switched to Bridged Adapter:* instead of forwarding, I changed the VM's network adapter from *NAT* to *Bridged*, which makes the VM request its own IP directly from my home router, just like a real device on the network. After restarting the VM and running 'ip a' again, I got a new address: `192.168.50.47` — a proper home network IP.

# Problem 2: Connection Refused

With the new bridged IP, I tried SSH again:
```bash
ssh labadmin@192.168.50.47
```
Result: *connection refused* (a different error than before — meaning the host could now reach the VM, but nothing was listening on port 22).

*Diagnosis:* checked the SSH service status directly on the VM:
```bash
sudo systemctl status ssh
```
Output: 'Unit ssh.service could not be found.' — SSH was never installed. The unattended install wizard apparently didn't include the OpenSSH package by default.

*Fix:*
```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl start ssh
sudo systemctl enable ssh
sudo systemctl status ssh
```

Confirmed it was now active and running:

![SSH service active and running](./screenshots/11-ssh-service-status.png)

## Success

Tried SSH one more time from PowerShell:
```bash
ssh labadmin@192.168.50.47
```

This time it prompted to trust the host's fingerprint ('yes'), asked for the password, and connected successfully:

![SSH connection refused, then successful after fix](./screenshots/12-ssh-connection-refused-then-success.png)

![Active SSH session into the server](./screenshots/13-ssh-session-active.png)

First successful remote login into a server I built myself, from my own machine, without ever touching the VM's console window directly.

---

# Step 6 – Post-Login Housekeeping

Once connected via SSH, I ran a standard update to pull any pending patches:

```bash
sudo apt update && sudo apt upgrade -y
```

![Running apt update and upgrade over SSH](./screenshots/14-apt-update-upgrade.png)

---

# What I Learned

| Problem | Root Cause | Fix |

| SSH connection timed out | VM was in NAT mode, isolated from my home network | Switched adapter to Bridged so the VM gets a real LAN IP |
| SSH connection refused (after fixing network) | OpenSSH server was never installed during setup | 'sudo apt install openssh-server -y', then started + enabled the service |

Beyond the fix itself, the bigger lesson was learning to read the difference between error messages - "timed out" and "connection refused" sound similar but point to two completely different problems (unreachable host vs. nothing listening on the port). Being able to tell those apart is a real diagnostic skill.

I also learned:
- '127.0.0.1' (loopback) vs. a real network interface address - and why only one of them is useful for remote connections
- The practical difference between VirtualBox's NAT and Bridged network modes
- How to check, start, and enable a systemd service (`systemctl status/start/enable`)
- That `sudo` always prompts for *your own* password, not a separate admin password


# Next Steps

- Practice more day-to-day Linux commands (file permissions, process management, log review) now that SSH access is working
- Try [OverTheWire: Bandit](https://overthewire.org/wargames/bandit/) to build more command-line reps
- Eventually connect this server to other lab VMs (Kali, DVWA) as a target machine
