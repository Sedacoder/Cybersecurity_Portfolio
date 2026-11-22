- Today we are exploiting DC-1, a vulnerable CTF machine from my Kali Attacker machine.

# Phase 1: Reconnaisance
- I start by ensuring they are on the same network by placing them both on NAT or Bridged Adapter or Host-Only Adapter.

## Step 1: Discovering the Target IP
- Running commands such as
`sudo arp-scan -l`
- sends arp requests to every device on my local network(subnet) hence the -l

`sudo netdiscover -r <IP address>`
- Does active and passive ARP recon in a live updating interface.

`sudo nmap -sN 192.168.100.0/24`
- Runs an nmap ping scan. The *Limitation* is that most modern OS or static firewalls drop ICMP pings by default so it is  not reliable.


## Step 2: Port & Service Enumeration(nmap)
`sudo nmap -sC -sV -oN nmap_dc1.txt <TARGET_IP>`
- Scans the 1000 most common TCP ports and determines the exact software and version numbers running on any open ports.
- *-sC* - Enables default script scanning. It runs 'safe' scripts to gather more context.
*-sV* - Enables version detection, probing open ports to determine what app is there and the version listening.
*-oN dc1_nmap.txt* - Outputs the scan results in a normal human-readable file(dc1_nmap.txt). *ALWAYS save your scan data so you have persistent documentation in CTFs and in pen tests.*

Alt commands:
`sudo nmap -p- -T4 <TARGET_IP>`
- Scans all 65,535 TCP ports(-p-)
- Used to ensure in case a CTF creator hides a service on a non-standard port eg.8080,31337 you will not miss it

At the end of Phase 2, you should see 3 primary open ports
<Insert screenshot>
1. Port 22(SSH) - Running OpenSSH will be useful later if we obtain the credentials.
2. Port 80(HTTP) - Running Apache and hosts a Drupal app
3. Port 111(RPC)

## Step 3: Web App fingerprinting and enumeration

### 1. Passive CMS & Version fingerprinting
- We start by inspecting the HTTP headers and default information files that Content Management Systems(CMS) often expose.

`curl -s http://<TARGET_IP>/CHANGELOG.txt | head -n 20`
- What it does: Fetches the CHANGELOG.txt file directly from the web root and displays the first 20 lines in your terminal. In Drupal installations, this file is included by default and explicitly states the exact minor release version (e.g., Drupal 7.31).

- Where it is used: Passive web reconnaissance and vulnerability auditing. Knowing the exact minor version lets you look up public security advisories (CVEs) to see if the software is unpatched.

Alternative Commands:

`whatweb http://<TARGET_IP>`: Scans the target URL and identifies web technologies, CMS versions, server headers, and JavaScript libraries in a single summary line.

`curl -I http://<TARGET_IP>`: Fetches only the HTTP response headers (such as X-Generator: Drupal 7) without downloading the entire webpage body.

### 2. Automated Directory & File Enumeration(gobuster)
`mgobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40`(Overwhelms the VM. Explain accordingly, DoS the system)

- Iterates through a wordlist of common directory and file names, sending HTTP requests for each entry(e.g., /admin, /user, /includes) and reporting valid HTTP status codes.
*dir* - Specifies directory and file enumeration mode
*-u* - Sets target URL.
*-w* - Specifies the path to the wordlist on Kali linux.
*-t 40* - Tuns 40 threads concurrently to speed up the scan 

- Used in Web app assessments and CTF reconnaisance to uncover unlinked endpoints and admin interfaces.


> *Tuning Enumeration Speed to Target Capacity*: Aggressive threading (-t 40+) against resource-constrained virtual machines or legacy PHP applications can exhaust server worker pools, triggering 500 Internal Server Error responses and false-negative timeouts. Always start directory enumeration with a concise wordlist (common.txt) and conservative thread counts (-t 10) before scaling up.

`gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt -t 10 -b 404,500`
- *common.txt* - Smaller wordlist of ~4600 entries. Helps finish faster
- *-t 10* - Keeps concurrency low enough so Apache/PHP won't choke.
*-b 404,500* - Blackilsts 404 and 500 errors from cluttering your terminal output


Alternative commands:
`ffuf -u http://<TARGET_IP>/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt`

`dirb http://<TARGET_IP>`

# Phase 2:Vulnerability Analysis & Exploitation
## Step 1: Cross-referencing Software versions with known CVEs (searchsploit)
`searchsploit drupal 7`
- Searches your local offline copy of *Exploit-DB*, a database of published exploits, proof-of-concept scripts or security advisories.
- Kali Linux ships with a local mirror of Exploit-DB(/usr/share/exploitdb)
- Used in standard vulnerability research.

> Why offline exploit databases matter: In real-world enterprise assessments, external internet access from testing environments may be restricted by strict firewalls or corporate proxies. Maintaining a local mirror of Exploit-DB via searchsploit ensures continuous vulnerability research capabilities regardless of network connectivity.


### Step 2: Mechanics of CVE-2014-3704 (Drupalgeddon)
Once software fingerprinting confirmed the target was running **Drupal 7** (versions prior to 7.32), vulnerability research via `searchsploit` identified **CVE-2014-3704**, commonly known as "Drupalgeddon."

#### Technical Root Cause
* **Vulnerable Component:** Drupal's database abstraction layer (`includes/database/database.inc`), specifically the `expandArguments()` function.
* **The Flaw:** When constructing parameterized queries with `IN` clauses, `expandArguments()` correctly parameterized query *values* but failed to sanitize associative array *keys*.
* **Exploitation Vector:** Unauthenticated endpoints (such as user login or password reset forms) accept POST parameters that are passed directly into database queries. By supplying a crafted associative array with malicious keys, an attacker can inject arbitrary SQL commands.
* **Impact:** Because PHP's PDO MySQL driver permitted stacked queries in these configurations, an unauthenticated attacker could execute raw SQL statements—such as inserting a new administrative user directly into the database.

#### How we knew the vulnerability to choose
Impact Alignment (The "Goal-Forward" Filter):
Discard DoS (Denial of Service): Crashing the web server takes the target offline, preventing you from interacting with it or retrieving flags.

Discard Client-Side Flaws (XSS): Cross-Site Scripting requires a victim (like an administrator) to view malicious input in their browser. On a standalone lab VM with no active simulated users browsing the site, XSS is a dead end.

Prioritize Foothold & Privilege Elevation (RCE or SQLi): You need an unauthenticated vulnerability that grants server-side execution or administrative access.

Version Precision:

Our fingerprinting proved the target is running Drupal 7 (specifically pre-7.32).

While later exploits like Drupalgeddon 2 (CVE-2018-7600) also affect Drupal 7, CVE-2014-3704 matches the exact vintage of DC-1 (released around 2014) and targets the core database abstraction layer unique to early Drupal 7 releases.

### Step 3: Exploit Validation & Foothold Execution
#### 1. Inspect the Exploit script
>  Safe Execution of Third-Party PoCs: Security analysts should never blindly execute raw scripts downloaded from the internet. Inspecting PoC code prevents unexpected side effects, uncovers hardcoded backdoor credentials created by the script, and deepens your technical understanding of how application parameters map to backend vulnerabilities.

Before executing external proof-of-concept scripts, a code review of `34992.py` was conducted to understand its implementation and safety:

* **Command-Line Parsing (`optparse`):** The script utilizes the legacy Python 2 `optparse` library to accept target URL arguments (`-t`), allowing custom administrative usernames and passwords to be defined or defaulted to standard values.
* **Payload Construction:** The script crafts an HTTP POST request targeting Drupal's authentication form. It injects malicious SQL statements through associative array keys rather than input values, exploiting the `expandArguments()` flaw in Drupal's database abstraction layer.
* **Execution Vector:** By leveraging PHP's PDO MySQL driver allowance of stacked queries, the injected SQL commands insert a new administrative record directly into the database users table.

#### 2. Executing and observing the script

`python2 34992.py -t <TARGET_IP>`

#### 3. Vailidating the foothold


>Triage and Selection in Vulnerability Analysis: When faced with multiple potential exploits for a software stack, analysts must filter by exploit reliability, target version precision, and access requirements (unauthenticated remote vs. authenticated local). Choosing a direct SQL injection that creates an administrative user provides a clean, verifiable foothold without destabilizing the target host.

## Phase 4: Post-Exploitation, System Access & Flag Recovery

### Objective
Transition from web-level administrative privileges (established via CVE-2014-3704) into system-level file access and configuration analysis to retrieve structural flags.

---

### 1. Tactical Pivot: Moving to Framework-Based Execution

During initial post-exploitation attempts, manual custom module deployment and legacy execution scripts presented environment-specific friction (e.g., PHP version warnings, routing 404 errors, and missing helper dependencies). 

* **The Pivot:** To maintain assessment momentum and ensure reliable execution without destabilizing the target host, the workflow shifted to the **Metasploit Framework**.
* **Rationale:** Metasploit abstracts protocol handling and session management, providing standardized exploitation modules that reliably bridge web-layer vulnerabilities into interactive command shells or file-retrieval mechanisms.

---

### 2. Executing Metasploit's Drupalgeddon Module

The automated exploit framework was utilized to target the application's unauthenticated SQL injection vulnerability directly:

1. **Module Selection:** Loaded *exploit/unix/webapp/drupal_drupageddon2*.
`use exploit/unix/webapp/drupal_drupadeggon`
2. **Configuration:** Targeted the lab IP (`RHOSTS`) and established target parameters.
`set RHOSTS = <TARGET_IP>
3. **Execution:** Successfully fired the payload, confirming stability and gaining system-level interaction with the underlying OS.
`exploit`
---

### 3. Flag 1 Recovery & Configuration Discovery

With stable execution established, system files were targeted to locate the first core lab objective:

* **Target File:** `/var/www/flag1.txt`
* **Finding:** Successfully extracted Flag 1, which contained a critical structural hint:
  > *"Every good CMS needs a config file - and so do you."*
* **Significance:** This pointed directly toward Drupal's core application configuration architecture as the next vector for credential harvesting.

---

### 4. Flag 2 & Database Credential Extraction

Following the hint provided by Flag 1, inspection shifted to Drupal's central configuration file:

* **Target File:** `/var/www/sites/default/settings.php`
* **Findings:**
  * **Flag 2:** Recovered successfully from the header comments of the configuration file.
  * **Database Credentials:** Extracted the MySQL connection array (`$databases`), revealing the application's database name (`drupaldb`), database user (`dbuser`), and plaintext password (`R0ck3t`).
* **Why This Matters:** Harvesting plaintext database credentials from configuration files is a primary objective in internal network penetration tests. These credentials often grant lateral movement capabilities across database services (Port 3306) and provide deeper insight into application data architecture.

> Now I just discovered flag3 inside the content page of the drupal site. I cannot really explain much to it😭

## Phase 5: Privilege Escalation & Flag 4
- I verified my privilege by running `whoami` and discovered I was **root**
- Therefore upon looking inside the directory I found the final flag

**CTF 001 DONE!!**