# Lab 01 – Network & Traffic Analysis

## Overview

This lab uses *Wireshark* and *tcpdump* to capture and analyze real network traffic on my Kali Linux VM. The goal was to understand what's actually moving across a network at the packet level — not just theory, but seeing it live on screen.

This lab genuinely changed how I think about network security. Watching plain text HTTP traffic scroll by in Wireshark and being able to read actual webpage HTML inside a raw packet made concepts like credential interception and network sniffing click in a way that no guided room ever did.

---

## 🎯 Objectives

- Launch Wireshark from the terminal and select a capture interface
- Capture live network traffic and filter by protocol
- Read plain text HTTP data from a captured packet
- Analyze DNS queries and understand what they reveal
- Use tcpdump from the command line to capture and filter traffic
- Save captures to `.pcap` files and read them back
- Understand the real-world security implications of unencrypted traffic

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| Wireshark 4.6.6 | GUI packet capture and analysis |
| tcpdump | Command-line packet capture |
| Firefox | Generating HTTP traffic to capture |
| neverssl.com | Intentionally unencrypted HTTP site |

---

## 🖥️ Environment

| Item | Details |
|------|---------|
| Attack Platform | Kali Linux 2026.1 VM in VirtualBox |
| Network Interface | eth0 (NAT mode) |
| Host IP | 10.0.2.15 |

---

## ⚙️ Step 1 – Boot Kali and Launch Wireshark

Started the Kali Linux VM and logged in:

![Kali Linux desktop on boot](./screenshots/01-kali-desktop-boot.png)

Launched Wireshark from the terminal:

```bash
sudo wireshark
```

The Wireshark welcome screen shows all available network interfaces. The one we care about is *eth0*:

![Wireshark interface selection screen showing eth0](./screenshots/02-wireshark-interface-selection.png)

---

## 🔬 Exercise 1 – Capture Live Traffic with Wireshark

Double-clicked *eth0* to start capturing. Wireshark immediately started filling with packets:

![Wireshark live capture in progress](./screenshots/03-wireshark-live-capture.png)

While capturing, opened Firefox and browsed to `http://neverssl.com` — a site that deliberately never uses HTTPS:

![Wireshark capturing traffic while browsing neverssl.com](./screenshots/04-wireshark-neverssl-capture.png)

---

## 🔬 Exercise 2 – Reading HTTP Packets

Applied an `http` filter. Out of 386 total packets, only 4 were HTTP:

| Packet | Info | What it means |
|--------|------|---------------|
| 26 | `GET /success.txt?ipv4 HTTP/1.1` | Kali checking "do I have internet?" in the background |
| 28 | `HTTP/1.1 200 OK (text/plain)` | Server confirming yes |
| 106 | `GET / HTTP/1.1` | Browser requesting the neverssl.com homepage |
| 110 | `HTTP/1.1 200 OK (text/html)` | Server sending back the actual webpage |

Clicked on packet 110 and expanded the packet data — the raw HTML of neverssl.com was completely readable in plain text:

![HTTP filter showing 4 packets with full HTML visible in packet data pane](./screenshots/05-wireshark-http-filter-packet-data.png)

```
<html>  <head>
<title> NeverSSL - Connecting...
</title>
```

This is the key takeaway of the whole lab. Anyone on the same network running Wireshark could read this. On public WiFi, that could be login credentials or personal data — anything sent over HTTP instead of HTTPS. This is exactly why HTTPS exists.

---

## 🔬 Exercise 3 – DNS Traffic Analysis

Changed the filter to `dns`. Every domain lookup the VM made appeared — including things never consciously visited:

- `example.org` — OS background connectivity check (reserved domain, never a real site)
- `cloudflare-dns.com` — Kali's DNS resolver, Cloudflare sees every domain lookup
- `detectportal.firefox.com` — Firefox automatically checking if it's behind a captive portal
- `ipv6only.arpa` — VM testing whether the network supports IPv6

Even sitting idle, a machine is constantly contacting dozens of servers in the background. Security analysts watch DNS traffic to spot malware — suspicious domain lookups that don't fit normal patterns are often the first sign something is wrong.

---

## 🔬 Exercise 4 – tcpdump: Command Line Packet Capture

Closed Wireshark and moved to the terminal. tcpdump does the same job but entirely from the command line:

```bash
sudo tcpdump -i eth0
```

![tcpdump starting up on eth0](./screenshots/06-tcpdump-start.png)

Traffic immediately started scrolling in raw text form:

![tcpdump live traffic scrolling](./screenshots/07-tcpdump-traffic-scrolling.png)

![tcpdump full output showing IP conversations](./screenshots/08-tcpdump-full-output.png)

### Filter to port 80 (HTTP only):

```bash
sudo tcpdump -i eth0 port 80
```

Browsed to `http://neverssl.com` while this ran — much cleaner output showing only the HTTP conversation:

![tcpdump filtering port 80 with neverssl.com open in browser](./screenshots/09-tcpdump-port80-neverssl.png)

---

## 🔬 Exercise 5 – Save and Read a PCAP File

Saved the capture directly to a file:

```bash
sudo tcpdump -i eth0 port 80 -w ~/Lab-Captures/tcpdump-http-captures.pcap
```

Read the saved capture back from the command line:

```bash
sudo tcpdump -r ~/Lab-Captures/tcpdump-http-captures.pcap
```

![Reading the pcap file back with tcpdump](./screenshots/10-tcpdump-read-pcap.png)

![HTTP GET request visible in the pcap replay](./screenshots/11-tcpdump-pcap-http-get.png)

![Full pcap output complete](./screenshots/12-tcpdump-pcap-complete.png)

The saved capture replayed perfectly — same HTTP GET requests and 200 OK responses from the live capture, now permanently saved for later analysis.

---

## 🐛 One Thing That Went Wrong

Hit a case sensitivity error reading the pcap file:

```bash
sudo tcpdump -r ~/lab-captures/tcpdump-http-captures.pcap
tcpdump: /home/kali/lab-captures/tcpdump-http-captures.pcap: No such file or directory
```

**Cause:** Linux is case sensitive. The folder was named `Lab-Captures` but I typed `lab-capture`.

**Fix:** ran `ls ~/` to see the exact folder name, then used the correct capitalization and missing `s`.

*Lesson:* In Linux, every character in a file path must match exactly — case, spelling, hyphens, everything. One wrong character and the command fails.

---

## 📝 What I Learned

| Concept | What It Means in Practice |
|---------|--------------------------|
| HTTP vs HTTPS | HTTP traffic is plain text — anyone on the network can read it |
| DNS reveals intent | DNS queries show every domain a machine contacts, even without reading packet content |
| Background traffic | Machines constantly talk to servers without user interaction |
| Wireshark vs tcpdump | Same job — Wireshark for visual analysis, tcpdump for servers with no GUI |
| PCAP files | Saved captures can be replayed, shared, and loaded into other tools |
| Linux case sensitivity | File paths are exact — wrong case = file not found |

---

## 🔒 Real World Security Relevance

- *Network sniffing:* What I did today is exactly what an attacker does on unsecured WiFi — capture traffic and read credentials from HTTP sites
- *Threat hunting:* Security teams watch DNS queries continuously for unknown domains — often the first sign of malware
- *Incident response:* PCAP files are submitted as evidence and used to reconstruct what happened during a breach
- *Why HTTPS matters:* If neverssl.com had used HTTPS, that bottom pane would have shown gibberish instead of readable HTML

---

## ⏭️ Next Steps

- Run Wireshark on home network with Bridged adapter to see all devices
- Learn Wireshark's "Follow TCP Stream" feature to read full conversations
- Move on to [Lab 02 – Vulnerability Scanning](../02-vulnerability-scanning/README.md)
