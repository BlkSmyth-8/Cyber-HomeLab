# Lab 00 – Kali Linux VM Setup

## Overview

This was my very first virtual machine - ever. I installed *Kali Linux 2026.1* inside Oracle VirtualBox on my Lenovo ThinkPad P14s running Windows 11. Kali Linux is the industry-standard operating system for penetration testing and security research, coming pre-loaded with hundreds of tools used by real security professionals.

Unlike most Linux distros where you install from an ISO, Kali offers a pre-built VirtualBox image — meaning the OS is already installed and configured inside a compressed file you just extract and import. This is the approach I used here.

This was also my introduction to the Linux command line. I noticed pretty quickly that having a Linux command reference cheat sheet pulled up on the side helped a lot — I was learning as I went.


# Objectives

- Download and extract the Kali Linux VirtualBox image
- Import and configure the VM in VirtualBox
- Successfully boot into Kali Linux for the first time
- Run the first system update
- Get familiar with the Kali desktop and pre-installed tools


# Environment

| Item | Details |

| Host Machine | Lenovo ThinkPad P14s (Windows 11) |
| Hypervisor | Oracle VirtualBox |
| Guest OS | Kali Linux 2026.1 (VirtualBox pre-built image) |
| RAM Allocated | 8192 MB |
| CPUs | 4 |
| Video Memory | 128 MB |
| Network | NAT (default) |


# Step 1 – Download Kali Linux & 7-Zip

Kali Linux offers pre-built VM images specifically for VirtualBox - no manual installation needed. The image comes as a compressed `.7z` archive.

1. Go to [https://www.kali.org/get-kali/#kali-virtual-machines](https://www.kali.org/get-kali/#kali-virtual-machines)
2. Download the **VirtualBox 64-bit** image
3. Download and install **7-Zip** from [https://www.7-zip.org](https://www.7-zip.org) to extract the archive (Windows can't open `.7z` files natively)

> The download is large — expect it to take a while depending on your connection speed.


# Step 2 – Extract the Archive

Once downloaded, right-click the `.7z` file → *7-Zip* → *Extract to folder*.

The archive contains roughly 15GB of files and takes around 25 minutes to extract — this is normal since it's unpacking a full pre-installed operating system:

![Extracting the Kali Linux archive — 15.1GB, roughly 25 minutes](./screenshots/03-extracting-archive.png)

The extracted folder contains the `.vdi` disk file (the virtual hard drive with Kali already installed on it) and a `.vbox` configuration file.

---

# Step 3 – Import into VirtualBox

Rather than creating a new VM from scratch, you can import the pre-built image directly:

1. Open VirtualBox
2. Double-click the `.vbox` file inside the extracted folder — VirtualBox imports it automatically
3. The VM appears in your VirtualBox list ready to configure

---

# Step 4 – Configure VM Settings

Before first boot, I opened *Settings* to review and adjust the hardware allocation:

![VM settings — System tab showing RAM and CPU configuration](./screenshots/04-vm-settings.png)

| Setting | Value | Why |

| Base Memory | 8192 MB | Kali runs a full desktop environment with lots of tools — more RAM means smoother performance |
| Processors | 4 | Allows multiple tools to run simultaneously without slowdown |
| Video Memory | 128 MB | Maximum — gives the best display performance for the GUI |
| Network | NAT | Default — gives internet access for downloading tools and updates |

---

## ✅ Step 5 – First Boot

With settings confirmed, I clicked **Start**. The VM booted directly into the Kali Linux desktop — no installer needed since the image came pre-installed:

![Kali Linux desktop on first boot](./screenshots/05-kali-first-boot.png)

![Kali Linux desktop running in VirtualBox](./screenshots/06-kali-desktop.png)

The Kali desktop (called XFCE) is clean and minimal. The taskbar at the top shows workspaces 1-4, and the pre-installed tools are accessible from the Applications menu.

> *Default credentials for the pre-built image:* username `kali`, password `kali`. Change these on any real deployment — for a local isolated lab VM it's acceptable to leave as-is.


# Step 6 – First System Update

One of the first things to do on any new Linux system is pull the latest updates. I opened a terminal and ran:

```bash
sudo apt update && sudo apt upgrade -y
```

This downloads and installs all available updates for Kali and its 600+ pre-installed tools. With that many packages it takes a while:

![apt upgrade running — downloading hundreds of packages](./screenshots/07-kali-update-running.png)

![apt upgrade continuing](./screenshots/08-kali-update-running2.png)

The progress bar at the bottom of the terminal shows the download progressing across all packages. Once complete, Kali is fully up to date.

---

# One Thing That Went Wrong

Looking at the VM Details panel early on, I noticed this under Storage:

```
SATA Port 0: kali-linux-2026.1-virtualbox-amd64.vdi (Normal, Inaccessible)
```

The *Inaccessible* status appeared because VirtualBox was looking for the `.vdi` file in a temp location that no longer existed after extraction. This is a common first-timer issue - VirtualBox tracks exact file paths, so if the file moves after import, it loses track of it. It resolved once the correct path was set.

Good early lesson: don't move VM files around after importing them.

---

# What I Learned

- Kali Linux is Debian-based and comes with 600+ pre-installed security tools
- Pre-built VM images skip the installer entirely, you go straight to a working desktop
- `sudo apt update && sudo apt upgrade -y` is the standard Debian/Ubuntu update command — `update` refreshes the package list, `upgrade` installs the newer versions
- VirtualBox tracks exact file paths for `.vdi` disk files — moving them after import causes Inaccessible errors
- Having a Linux command reference open while working is not a crutch, it's just smart when you're starting out

---

# Next Steps

With Kali running, this VM becomes the *attack platform* for all other labs in this portfolio. Next up: setting up an Ubuntu Server VM for command-line fundamentals, then moving into network analysis, vulnerability scanning, and web app hacking.
