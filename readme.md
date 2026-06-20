
# Kioptrix.level1 using Metasploit

Kioptrix Level 1 is a beginner-friendly vulnerable Linux machine originally created for learning penetration testing and exploitation techniques. The goal is to identify exposed services, enumerate potential vulnerabilities, gain initial access to the target system, and ultimately obtain root privileges.

This machine is widely used by aspiring penetration testers to practice a complete attack lifecycle, including reconnaissance, service enumeration, vulnerability assessment, exploitation, and privilege escalation. In this writeup, the compromise is achieved using the Metasploit Framework (in this write-up) to exploit a vulnerable network service and gain administrative access to the target host.

Link: https://www.vulnhub.com/entry/kioptrix-level-1-1,22/


## Methodology and Mindset

This writeup is intended to demonstrate the thought process of a penetration tester rather than simply providing a sequence of commands that lead directly to root access.

One of the reasons Kioptrix Level 1 remains a popular practice machine is that it contains multiple attack paths that can ultimately result in full system compromise. Rather than immediately selecting a known exploit, the goal of this assessment is to approach the target as if no prior knowledge exists and allow the information gathered during enumeration to guide the next steps.

Throughout the engagement, we will follow a structured penetration testing methodology:

1. **Reconnaissance** – Identify the target and discover exposed services.
2. **Enumeration** – Gather as much information as possible about those services, versions, and potential weaknesses.
3. **Vulnerability Assessment** – Analyze the collected information to identify possible attack vectors.
4. **Exploitation** – Use the most promising avenue to gain initial access.
5. **Privilege Escalation** – Escalate privileges until full administrative (root) access is achieved.
6. **Post-Exploitation** – Verify the compromise and understand the impact.

At each stage, different tools will be used to answer a single question:

> **"What opportunities does the information gathered so far present?"**

Instead of treating tools such as Nmap, Nikto, Metasploit, searchsploit, or manual enumeration techniques as solutions, they should be viewed as methods of discovering additional information. The findings from one phase will determine which tool or technique is used next.

This approach mirrors real-world penetration testing, where success depends less on knowing a specific exploit and more on understanding how to systematically identify, investigate, and validate potential attack paths.

The objective of this writeup is therefore not only to compromise the Kioptrix Level 1 machine, but also to illustrate the decision-making process that leads from initial reconnaissance to root access.

## Pre-requiste Information

The following information are provided before for the readers to have an easier grasp of the respective machines:

Target IP: 192.168.126.129\
Host IP: 192.168.126.128
# Reconnaissance

## Step 1: Host Discovery

The first step was to identify the IP address of the target machine on the local network. This can be accomplished using tools such as **Netdiscover** or **Nmap**.

### Option 1: Using Netdiscover

```bash
sudo netdiscover -r 192.168.126.0/24
```
![NetdiscoverScan](https://github.com/WNobsi/kioptrix.level1-using-Metaploit-/blob/61b1e8da44e88428b31cd401e964ad8088326fea/images/scan_netdiscover.png)

### Option 2: Using Nmap Ping Sweep

```bash
nmap -sn 192.168.126.0/24
```

### Results

After scanning the subnet, the target machine was identified with the following IP address:

```text
192.168.126.129
```

This IP address will be used throughout the remainder of the assessment for enumeration and exploitation.

## Step 2: Scanning for open ports

After we have identified the IP for the victim machine, we head to Nmap again to scout for open ports.

```bash
nmap -sS -T4 -p- -A 192.168.126.129
```

![scan_nmap_ports](placholder.png)

### Results

#### Scan Options

| Flag  | Description                                                                |
| ----- | -------------------------------------------------------------------------- |
| `-sS` | TCP SYN Scan                                                               |
| `-T4` | Faster scan timing                                                         |
| `-p-` | Scan all 65,535 TCP ports                                                  |
| `-A`  | Aggressive scan (OS detection, version detection, NSE scripts, traceroute) |

---

#### Scan Summary

* Host is alive
* 65,529 TCP ports closed
* 6 TCP ports open

| Port  | Service | Version            |
| ----- | ------- | ------------------ |
| 22    | SSH     | OpenSSH 2.9p2      |
| 80    | HTTP    | Apache 1.3.20      |
| 111   | RPCBind | RPC Service        |
| 139   | SMB     | Samba              |
| 443   | HTTPS   | Apache 1.3.20 SSL  |
| 32768 | Status  | RPC Status Service |

---

## Port Analysis

### Port 22 - SSH

```text
22/tcp open ssh OpenSSH 2.9p2 (protocol 1.99)
```

#### Observations

* OpenSSH version 2.9p2
* Supports both SSHv1 and SSHv2
* SSHv1 is deprecated and considered insecure

#### Potential Opportunities

* Username enumeration
* Credential attacks (if valid credentials are discovered later)
* Research for historical vulnerabilities affecting this version

---

### Port 80 - HTTP

```text
80/tcp open http Apache httpd 1.3.20
```

#### Observations

* Apache 1.3.20 running on Red Hat Linux
* Default Apache test page present
* TRACE method enabled

### Interesting Findings

```text
Potentially risky methods: TRACE
```

Default web pages often indicate:

* Minimal configuration
* Hidden directories
* Backup files
* Administrative interfaces

### Port 111 - RPCBind

```text
111/tcp open rpcbind
```

#### Observations

RPC services are commonly associated with:

* NFS
* Legacy Linux services
* Remote procedure call frameworks

#### Potential Opportunities

* Information disclosure
* NFS share discovery
* Service enumeration

---

### Port 139 - SMB (Samba)

```text
139/tcp open netbios-ssn
Samba smbd (workgroup: MYGROUP)
```

#### Observations

* Samba service detected
* Workgroup: MYGROUP

#### Potential Opportunities

SMB services are frequently valuable during assessments because they may reveal:

* Shared folders
* Usernames
* Misconfigurations
* Anonymous access

---

### Port 443 - HTTPS

```text
443/tcp open ssl/https
Apache/1.3.20
```

#### Observations

* Same Apache version as HTTP
* SSLv2 supported
* Multiple weak SSL ciphers enabled

#### Interesting Findings

* Expired certificate
* Self-signed certificate
* Default hostname

These indicators suggest a very old or intentionally vulnerable environment.

---

### Port 32768 - RPC Status

```text
32768/tcp open status
```

#### Observations

RPC Status Service detected:

```text
status (RPC #100024)
```

#### Potential Opportunities

* Additional RPC enumeration
* Discovery of related services
* NFS-related information gathering

---

### Additional Nmap Findings

#### Operating System Detection

```text
Linux Kernel 2.4.9 - 2.4.18
```

#### Observations

* Extremely old Linux kernel
* Potentially vulnerable to historical kernel exploits
* Indicates a legacy system

---

### SSL Configuration

#### Supported Protocols

```text
SSLv2 Supported
```

### Weak Ciphers Detected

* SSL2_RC2_128_CBC_WITH_MD5
* SSL2_RC4_128_EXPORT40_WITH_MD5
* SSL2_DES_64_CBC_WITH_MD5
* SSL2_RC4_64_WITH_MD5
* SSL2_DES_192_EDE3_CBC_WITH_MD5

These ciphers are obsolete and no longer considered secure.

---


## Pentester's Assessment of NMAP scan

At this stage, the objective is not exploitation but identifying the most promising attack surfaces.

Based on the reconnaissance results, the likely order of investigation would be:

- SMB (Port 139)
- Web Services (Ports 80 and 443)
- RPC Services (Ports 111 and 32768)
- SSH (Port 22) 

The scan reveals multiple outdated services, suggesting there may be several valid paths to compromise the target. The next phase should focus on thorough enumeration of each exposed service before attempting exploitation.
# Enumerating HTTP and HTTPS (Port 80 and 443)

## Web Enumeration with Nikto

After identifying the available web services during the Nmap phase, the next step is to perform vulnerability and configuration analysis of the web server using Nikto.

```bash
nikto -h http://192.168.126.129
```

#### Target Information

| Field           | Value           |
| --------------- | --------------- |
| Target IP       | 192.168.126.129 |
| Port            | 80              |
| Platform        | Linux/Unix      |
| Web Server      | Apache/1.3.20   |
| SSL Module      | mod_ssl/2.8.4   |
| OpenSSL Version | OpenSSL/0.9.6b  |

---

#### Server Information

Identified the following server banner:

```text
Apache/1.3.20 (Unix) (Red-Hat/Linux)
mod_ssl/2.8.4
OpenSSL/0.9.6b
```

#### Initial Observations

All three major components appear significantly outdated:

| Component | Version |
| --------- | ------- |
| Apache    | 1.3.20  |
| mod_ssl   | 2.8.4   |
| OpenSSL   | 0.9.6b  |

This immediately suggests a legacy web stack that may contain publicly documented vulnerabilities.

---

### ETag Information Disclosure

```text
Server may leak inodes via ETags
```

#### Finding

The server exposes ETag values that may disclose:

* Inode numbers
* File sizes
* Modification timestamps

#### Associated Vulnerability

```text
CVE-2003-1418
```

#### Impact

This information can help an attacker gather details about the underlying filesystem and server structure.

---

### Outdated Apache Version

```text
Apache/1.3.20 appears to be outdated
```

#### Observations

The detected version is significantly older than supported Apache releases.

Nikto further reports historical vulnerabilities affecting Apache 1.3.x including:

* Remote Denial of Service vulnerabilities
* Local buffer overflow vulnerabilities
* Potential code execution conditions in vulnerable configurations

#### Assessment

The presence of Apache 1.3.20 indicates a potentially vulnerable web server that warrants further investigation.

---

### mod_ssl 2.8.4 Analysis

#### High-Value Finding

One of the most interesting results from the Nikto scan is the presence of:

```text
mod_ssl/2.8.4
```

```text
mod_ssl 2.8.7 and lower are vulnerable to a remote buffer overflow which may allow a remote shell.
```

#### Why This Matters

The installed version:

```text
mod_ssl 2.8.4
```

Historically, older versions of mod_ssl suffered from multiple security issues, including buffer overflow vulnerabilities that could potentially lead to:

* Remote code execution
* Arbitrary command execution
* Remote shell access
* Full server compromise

#### Pentester's Perspective

When encountering an outdated Apache installation, attention should immediately shift toward the SSL implementation.

The combination of:

```text
Apache 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

represents a historically significant attack surface and is often one of the first areas security researchers investigate on legacy Linux systems.

This finding should be considered one of the most promising avenues discovered during web enumeration.

---

### OpenSSL 0.9.6b Analysis

Nikto identified:

```text
OpenSSL/0.9.6b appears to be outdated
```

#### Observations

The detected OpenSSL version is extremely old and predates many modern security improvements.

#### Potential Risks

Older OpenSSL versions may be susceptible to:

* Memory handling vulnerabilities
* Weak cryptographic implementations
* SSL/TLS protocol weaknesses
* Information disclosure vulnerabilities

#### Assessment

Combined with the outdated Apache and mod_ssl versions, this significantly increases the overall risk profile of the web server.

---

### Missing Security Headers

Nikto reported several missing security headers:

```text
Content-Security-Policy
Permissions-Policy
Strict-Transport-Security
Referrer-Policy
X-Content-Type-Options
```

#### Impact

The absence of these headers may increase exposure to:

* Content injection attacks
* MIME-type confusion attacks
* Clickjacking-related issues
* Information leakage through referrer headers

While common on older systems, their absence further highlights the age of the server configuration.

---

### HTTP Methods

Nikto identified the following allowed methods:

```text
GET
HEAD
OPTIONS
TRACE
```

#### Important Finding

```text
TRACE Method Enabled
```

Nikto reports:

```text
HTTP TRACE method is active and replies with suggestions that the host is vulnerable to XST.
```

#### Impact

TRACE can potentially be abused for:

* Cross-Site Tracing (XST)
* Header disclosure
* Debugging information leakage

Although not always directly exploitable, TRACE is generally considered unnecessary and should typically be disabled.

---

## Pentester's Assessment of Nikto scan

The Nikto scan paints a picture of a heavily outdated web server stack running:

```text
Apache 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

The most significant findings include:

- Outdated Apache 1.3.20 installation
- Outdated OpenSSL 0.9.6b implementation
- Potentially vulnerable mod_ssl 2.8.4 module
- TRACE method enabled
- Directory indexing enabled
- Accessible Apache documentation and default files
- Presence of a potentially interesting `test.php` page

From a penetration testing perspective, the **mod_ssl 2.8.4 finding stands out as the most promising discovery** because it falls within a historically vulnerable version range associated with remote buffer overflow vulnerabilities and potential remote code execution scenarios.

When we google for mod_ssl/2.8.4 exploit. 
We find:

- https://www.exploit-db.com/exploits/21671

- https://github.com/heltonWernik/OpenLuck


## Enumerating SMB (Port 139)

Now
