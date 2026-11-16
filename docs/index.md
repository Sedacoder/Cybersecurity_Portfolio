# Security Engineering & Penetration Testing Log

> **Core Focus:** Documenting hands-on labs, vulnerable infrastructure, and practical offensive security methodologies.

---

## Philosophy: Skills Over Badges
This repository serves as a technical log of my practical security training. Rather than chasing gamified leaderboards, platform points, or automated badges, my approach centers on **self-hosted labs, broken web applications, and real-world execution**. 

Every engagement and lab documented here follows a systematic methodology:
1. **Reconnaissance & Enumeration:** Deep-diving into service discovery, version analysis, and attack surface mapping.
2. **Exploitation:** Understanding root causes, researching vulnerabilities (such as CVEs and logical flaws), and executing controlled access.
3. **Post-Exploitation & Remediation:** Escalating privileges safely and writing actionable remediation steps to fix the underlying risk.

---

## What You'll Find in This Portfolio

### 1. Lab Write-Ups (`docs/labs/`)
Detailed walkthroughs of machines and web applications. Each write-up breaks down the exact hurdles faced, commands executed, and lessons learned:
* **VulnHub Machines:** Simulating enterprise and standalone environments (e.g., [DC-1 Walkthrough](labs/vulnhub/dc-1.md)).
* **Web Security:** Practical exploitation notes from broken web applications (PortSwigger Labs).

### 2. Toolkit & Scripts (`scripts/`)
Custom automation and utility scripts built to streamline repetitive reconnaissance or hardening tasks:
* **Offensive Utilities:** Small Python or Bash scripts written during lab workflows.
* **Defensive Engineering:** Scripts like my [Linux Hardening Framework](https://github.com/USERNAME/Cybersecurity_Portfolio/tree/main/scripts/linux-hardening) focused on baseline server security.

### 3. Quick References (`docs/toolkit/cheatsheet.md`)
A curated collection of one-liners, syntax guides, and post-exploitation commands used frequently during hands-on work.

---

## Current Status
* **Active Lab:** [DC-1 (VulnHub)](labs/vulnhub/dc-1.md) — CMS enumeration, Drupalgeddon exploitation, and privilege escalation.
* **Tooling Focus:** Nmap, Metasploit, Burp Suite, and Python automation.