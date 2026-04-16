---
title: "Nmap Scanning Techniques for Vulnerability Reconnaissance"
date: 2026-04-15
layout: default
description: "A practical walkthrough of using Nmap for network reconnaissance, service enumeration, and vulnerability detection as part of a penetration testing workflow."
---

# Nmap Scanning Techniques for Vulnerability Reconnaissance

## Overview

Nmap (Network Mapper) is the foundational tool for network reconnaissance in any penetration testing or vulnerability analysis workflow. Before running Metasploit exploits or Tenable scans, Nmap gives you the attack surface map — what hosts are alive, what ports are open, what services are running, and what versions those services are. This guide walks through how I use Nmap during the reconnaissance phase of assessments, from initial host discovery through targeted vulnerability detection.

---

## Step 1: Host Discovery

Before scanning ports, identify which hosts are alive on the network:

- Use ping sweeps to identify active hosts without triggering full port scans
- Combine multiple discovery techniques for accuracy across different network configurations
- Avoid DNS resolution on large subnets to reduce noise and time

**Common Host Discovery Commands:**
```bash
# ICMP ping sweep (requires root)
nmap -sn 192.168.1.0/24

# ARP discovery on local network (faster, more reliable on LAN)
nmap -sn -PR 192.168.1.0/24

# TCP SYN/ACK discovery (useful when ICMP is blocked)
nmap -sn -PS22,80,443 192.168.1.0/24

# Save live host list to file for later use
nmap -sn 192.168.1.0/24 -oG - | grep "Up" | awk '{print $2}' > live_hosts.txt
```

---

## Step 2: Port Scanning

With live hosts identified, scan for open ports to map the attack surface:

- Start with a fast top-ports scan, then follow up with a full port range on interesting targets
- Use SYN scan (`-sS`) for speed and stealth — does not complete the TCP handshake
- Adjust timing templates based on network stability and stealth requirements

**Port Scanning Techniques:**

| Scan Type | Flag | Use Case |
|-----------|------|----------|
| SYN Scan (Half-open) | `-sS` | Fast, stealthy — default for root |
| TCP Connect Scan | `-sT` | No root required |
| UDP Scan | `-sU` | Discover DNS, SNMP, TFTP |
| ACK Scan | `-sA` | Firewall rule mapping |
| Comprehensive | `-sS -sU` | Full TCP + UDP coverage |

```bash
# Fast scan of top 1000 ports
nmap -sS -T4 192.168.1.100

# Full port scan (all 65535 ports)
nmap -sS -p- -T4 192.168.1.100

# Top 1000 TCP + top 100 UDP
nmap -sS -sU --top-ports 1000 192.168.1.100
```

---

## Step 3: Service & Version Detection

Identify what software and version is running on each open port:

- Version detection is critical for matching findings against CVE databases
- The more accurate the version, the better your Tenable and Metasploit module matching will be
- Script scanning (`-sC`) runs default NSE scripts alongside version detection

```bash
# Service and version detection
nmap -sV 192.168.1.100

# Aggressive version detection (slower but more accurate)
nmap -sV --version-intensity 9 192.168.1.100

# Version + default scripts combined (common workflow starting point)
nmap -sV -sC 192.168.1.100

# Full enumeration — version, scripts, OS detection
nmap -A 192.168.1.100
```

**What to Look For:**
- Apache/Nginx versions — match against known CVEs
- OpenSSH version — identify outdated versions with known exploits
- SMB service on port 445 — flag for EternalBlue / PrintNightmare checks
- FTP with anonymous login allowed
- Outdated TLS/SSL on port 443

---

## Step 4: OS Detection

Fingerprint the operating system to refine your exploit selection:

- OS detection helps narrow down which Metasploit payloads and exploits are applicable
- Works best when Nmap has multiple open and closed ports to analyze

```bash
# OS detection (requires root)
nmap -O 192.168.1.100

# OS detection with version detection
nmap -O -sV 192.168.1.100

# Aggressive OS fingerprinting
nmap -O --osscan-guess 192.168.1.100
```

---

## Step 5: NSE Script Scanning

Nmap Scripting Engine (NSE) extends Nmap into a lightweight vulnerability scanner. Scripts are categorized by purpose and can be run selectively or in groups:

**Script Categories:**

| Category | Purpose |
|----------|---------|
| `default` | Safe, informational scripts run with `-sC` |
| `vuln` | Check for known vulnerabilities |
| `auth` | Test authentication and credentials |
| `brute` | Brute-force login attempts |
| `discovery` | Additional service enumeration |
| `safe` | Non-intrusive information gathering |

```bash
# Run default scripts
nmap -sC 192.168.1.100

# Run vulnerability detection scripts
nmap --script vuln 192.168.1.100

# Target specific services with relevant scripts
nmap --script smb-vuln* -p 445 192.168.1.100
nmap --script http-enum,http-headers,http-methods -p 80,443 192.168.1.100
nmap --script ssh-auth-methods -p 22 192.168.1.100
nmap --script ftp-anon,ftp-bounce -p 21 192.168.1.100

# Check for MS17-010 (EternalBlue) specifically
nmap --script smb-vuln-ms17-010 -p 445 192.168.1.100
```

---

## Step 6: Output & Documentation

Save scan results in multiple formats for reporting and tool integration:

- XML output integrates directly with Metasploit (`db_import`) and other tools
- Grepable output is useful for quick command-line parsing
- Always save raw output — you will reference it throughout the engagement

```bash
# Output to all formats simultaneously
nmap -sV -sC -oA scan_results 192.168.1.100
# Generates: scan_results.nmap, scan_results.xml, scan_results.gnmap

# XML only (for Metasploit import)
nmap -sV -oX scan_results.xml 192.168.1.100

# Import into Metasploit database
# In msfconsole:
# db_import scan_results.xml
```

---

## Step 7: Integrating Nmap Into the Broader Workflow

Nmap output feeds directly into the rest of your vulnerability analysis pipeline:

- **Tenable**: Use Nmap host discovery to define scan scope — export IP list and import into Nessus scan targets
- **Metasploit**: Import XML output with `db_import` to populate the `hosts` and `services` tables, then use `vulns` and `services` commands to plan exploits
- **Burp Suite**: Nmap identifies web servers (ports 80, 443, 8080, 8443) — feed those hosts into Burp for web application testing
- **Wazuh**: Cross-reference Nmap-discovered open ports against Wazuh agent data to identify unmonitored services

**Full Reconnaissance Workflow:**
```bash
# 1. Discover live hosts
nmap -sn 192.168.1.0/24 -oG live_hosts.gnmap

# 2. Full port scan on live hosts
nmap -sS -p- -iL live_hosts.txt -oA full_portscan -T4

# 3. Targeted service + script scan on open ports found
nmap -sV -sC -p 22,80,443,445 192.168.1.100 -oA targeted_scan

# 4. Vuln scripts on interesting findings
nmap --script vuln -p 445 192.168.1.100
```

---

## Tools & Resources

- **[Nmap Documentation](https://nmap.org/docs.html)** — Official reference for all flags and NSE scripts
- **[NSE Script Database](https://nmap.org/nsedoc/)** — Browse all available scripts by category
- **[Zenmap](https://nmap.org/zenmap/)** — GUI frontend for Nmap, useful for visualizing scan results
- **[Nmap Network Scanning Book](https://nmap.org/book/)** — Free online book by Nmap's creator
- **[SecLists](https://github.com/danielmiessler/SecLists)** — Wordlists and payloads to pair with Nmap brute-force scripts

---

## Important Ethical Considerations

- Only run Nmap scans against systems you own or have explicit written permission to test
- Aggressive scans (`-T5`, `--script vuln`) can cause service disruption on unstable systems — use caution in production environments
- Many organizations monitor for port scan signatures — coordinate with the client or SOC team before scanning
- Review local laws and engagement rules of engagement before any reconnaissance activity

---

## Final Thoughts

Nmap is the starting point for every engagement. Without a clear picture of what's on the network and what's running where, everything downstream — Tenable scans, Metasploit exploits, Burp Suite testing — is flying blind. A methodical Nmap reconnaissance phase produces a reliable asset inventory, accurate service data, and an early signal of high-value targets, setting up every subsequent phase of the assessment for success.
