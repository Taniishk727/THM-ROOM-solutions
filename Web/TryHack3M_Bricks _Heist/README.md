# TryHackMe — Bricks

## Overview

This write-up documents the approach used to solve the **Bricks** room.

The investigation involved network reconnaissance, WordPress enumeration, vulnerability identification, exploitation of a known WordPress vulnerability, obtaining a reverse shell, and investigating a suspicious service to identify a cryptocurrency mining process.

The main attack path was:

```text
Network Reconnaissance
        ↓
WordPress Enumeration
        ↓
CVE Identification
        ↓
Remote Code Execution
        ↓
Reverse Shell
        ↓
Service Enumeration
        ↓
Suspicious Service
        ↓
Miner Configuration
        ↓
Double Base64 Decoding
        ↓
Cryptocurrency Address
        ↓
Miner Identification
```

---

# 1. Initial Reconnaissance

I started by performing a port scan against the target machine using RustScan.

```bash
rustscan -a bricks.thm -- -A
```

The scan identified the following open ports:

```text
22/tcp
80/tcp
443/tcp
3306/tcp
```

The target IP identified during the scan was:

```text
10.48.189.133
```

### Findings

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 80   | HTTP    |
| 443  | HTTPS   |
| 3306 | MySQL   |

The presence of both HTTP and HTTPS indicated that a web application was likely an important part of the attack surface.

The exposed MySQL port was also noteworthy and could potentially become relevant during later enumeration.

---

# 2. Web Enumeration

The RustScan output also revealed information from `robots.txt`:

```text
http-robots.txt: 1 disallowed entry
|_/wp-admin/
```

The `/wp-admin/` path indicated that the target was running WordPress.

Navigating to the discovered path led to a WordPress login page.

### Reasoning

The discovery of `/wp-admin/` provided a strong indication that the application was WordPress.

At this point, rather than attempting to brute-force the login page immediately, I focused on identifying the WordPress version and looking for known vulnerabilities affecting the installed version.

---

# 3. Identifying the WordPress Vulnerability

Further enumeration revealed that the target was running:

```text
Bricks 1.9.5
```

The version provided a useful lead because known vulnerabilities could be searched for against the specific version.

The investigation identified:

```text
CVE-2024-25600
```

This vulnerability affects the Bricks WordPress theme and can be exploited to achieve remote code execution.

---

# 4. Exploiting CVE-2024-25600

A Python exploit for the identified CVE was used against the target:

```bash
python CVE-2024-25600.py -u https://bricks.thm
```

The exploit successfully provided a reverse shell.

### Reasoning

The vulnerability identification was based on the combination of:

```text
WordPress
     +
Bricks
     +
Version 1.9.5
     ↓
CVE-2024-25600
```

Instead of treating the WordPress login page as the only possible entry point, identifying the exact software version allowed the attack surface to be correlated with a known vulnerability.

Successful exploitation resulted in shell access to the target.

---

# 5. Retrieving the First Flag

After obtaining the reverse shell, I investigated the files available on the system.

The file:

```text
650c844110baced87e1606453b93f22a.txt
```

contained the first flag.

This confirmed that the initial access obtained through the WordPress vulnerability was successful.

---

# 6. Enumerating Running Services

With shell access established, the next objective was to identify anything unusual running on the system.

I listed the currently running services using:

```bash
systemctl list-units --type-service --state-running
```

### Reasoning

After gaining an initial shell, enumerating running services is useful for identifying:

* Unexpected services
* Custom services
* Persistence mechanisms
* Misconfigured services
* Suspicious processes

The service list contained a service named:

```text
ubuntu.service
```

This service stood out because its configuration did not appear to correspond to a normal Ubuntu service.

---

# 7. Investigating `ubuntu.service`

I inspected the service configuration:

```bash
systemctl cat ubuntu.service
```

The configuration contained:

```text
[Unit]
Description=TRYHACK3M

[Service]
Type=simple
ExecStart=/lib/NetworkManager/nm-inet-dialog
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

The most interesting line was:

```text
ExecStart=/lib/NetworkManager/nm-inet-dialog
```

### Analysis

The service was configured to execute:

```text
/lib/NetworkManager/nm-inet-dialog
```

This was suspicious because the executable was being launched as a persistent system service and configured to restart if it failed.

The service configuration therefore provided a strong indication that `nm-inet-dialog` deserved further investigation.

---

# 8. Investigating the Suspicious Process

I examined the contents of the NetworkManager directory:

```bash
ls /lib/NetworkManager/
```

The objective was to determine whether `nm-inet-dialog` was present and identify other files associated with it.

This investigation led to the discovery of a miner instance and its associated configuration.

---

# 9. Extracting the Miner Configuration

The discovered configuration file was:

```text
inet.conf
```

Opening the file revealed an encoded `id` value.

The value did not immediately appear to be readable text, so I moved to decoding it.

---

# 10. Decoding the ID

The encoded value was copied into CyberChef.

The investigation showed that the value was **double Base64 encoded**.

The decoding process was therefore:

```text
Encoded Value
      ↓
Base64 Decode
      ↓
Base64 Decode
      ↓
Decoded Address
```

After applying Base64 decoding twice, the encoded value was converted into an address.

### Reasoning

Recognizing multiple layers of encoding is important during malware and configuration analysis.

A Base64-looking value should not necessarily be assumed to require only one decoding operation. If the output of the first decoding step is still Base64-like data, another decoding layer may be present.

---

# 11. Identifying the Cryptocurrency Address

The decoded value represented a cryptocurrency address.

I searched for this address using Bitcoin.com and additional Internet searches.

The purpose was to identify information associated with the address and determine who or what was operating the mining infrastructure.

The investigation ultimately revealed the name associated with the cryptocurrency mining activity.

---

# Attack Chain

```text
Target
  |
  v
RustScan
  |
  +-- 22 SSH
  +-- 80 HTTP
  +-- 443 HTTPS
  +-- 3306 MySQL
  |
  v
robots.txt
  |
  v
/wp-admin/
  |
  v
WordPress
  |
  v
Bricks 1.9.5
  |
  v
CVE-2024-25600
  |
  v
Remote Code Execution
  |
  v
Reverse Shell
  |
  v
systemctl list-units
  |
  v
ubuntu.service
  |
  v
nm-inet-dialog
  |
  v
Miner
  |
  v
inet.conf
  |
  v
Encoded ID
  |
  v
Base64
  |
  v
Base64
  |
  v
Cryptocurrency Address
  |
  v
Miner Identification
```

---

# Commands Used

## Port Scanning

```bash
rustscan -a bricks.thm -- -A
```

## WordPress Enumeration

The scan identified:

```text
/wp-admin/
```

## Exploitation

```bash
python CVE-2024-25600.py -u https://bricks.thm
```

## Enumerating Running Services

```bash
systemctl list-units --type-service --state-running
```

## Inspecting the Suspicious Service

```bash
systemctl cat ubuntu.service
```

## Inspecting NetworkManager Files

```bash
ls /lib/NetworkManager/
```

---

# Tools Used

| Tool        | Purpose                                          |
| ----------- | ------------------------------------------------ |
| RustScan    | Port and service reconnaissance                  |
| Python      | Execute the CVE exploit                          |
| systemctl   | Enumerate and inspect running services           |
| CyberChef   | Decode the double Base64-encoded value           |
| Bitcoin.com | Investigate the recovered cryptocurrency address |

---

# Techniques Used

* Network reconnaissance
* Port scanning
* Web enumeration
* WordPress enumeration
* Vulnerability identification
* CVE exploitation
* Remote code execution
* Reverse shell
* Linux service enumeration
* Persistence analysis
* Suspicious service investigation
* Cryptocurrency miner investigation
* Base64 decoding

---

# Key Takeaways

## 1. Version Enumeration Can Lead Directly to Exploitation

Identifying the specific Bricks version was important because it allowed the application to be matched against a known vulnerability.

The sequence was:

```text
Identify Application
      ↓
Identify Version
      ↓
Search Known Vulnerabilities
      ↓
Identify CVE
      ↓
Test Exploit
```

## 2. Post-Exploitation Enumeration Is Important

Obtaining a shell was not the end of the investigation.

After gaining access, enumerating running services revealed an unusual service that led to the next stage of the room.

```bash
systemctl list-units --type-service --state-running
```

## 3. Suspicious Service Names Should Be Investigated

The service:

```text
ubuntu.service
```

appeared legitimate at first glance because of its generic name.

However, inspecting its configuration revealed:

```text
ExecStart=/lib/NetworkManager/nm-inet-dialog
```

This demonstrated why service names alone should not be trusted. The actual executable being launched is more important.

## 4. Persistence Can Reveal Malicious Activity

The service was configured with:

```text
Restart=on-failure
```

and launched the suspicious `nm-inet-dialog` executable.

This made the service configuration an important persistence-related artifact to investigate.

## 5. Encoded Configuration Data Requires Careful Analysis

The miner configuration contained an encoded ID.

The value required two Base64 decoding operations:

```text
Base64
   ↓
Base64
   ↓
Cryptocurrency Address
```

Recognizing repeated encoding layers allowed the investigation to continue toward identifying the mining infrastructure.

---

# Conclusion

The Bricks room demonstrated a complete progression from initial network reconnaissance to post-exploitation investigation.

The initial attack surface was identified using RustScan, which revealed the web services. Further enumeration identified a WordPress installation and the Bricks 1.9.5 theme. This led to CVE-2024-25600, which was exploited to obtain a reverse shell.

After gaining access, I shifted from exploitation to system-level enumeration. Running services were inspected, leading to the discovery of `ubuntu.service` and its suspicious `nm-inet-dialog` executable.

Further investigation uncovered a cryptocurrency miner and its configuration file. The encoded ID from the configuration was double Base64 encoded, and decoding it revealed a cryptocurrency address that could then be investigated to identify the associated mining activity.

The overall methodology was:

```text
Reconnaissance
    ↓
Enumeration
    ↓
Vulnerability Identification
    ↓
Exploitation
    ↓
Initial Access
    ↓
Post-Exploitation Enumeration
    ↓
Persistence Analysis
    ↓
Malware/Miner Investigation
    ↓
Encoded Data Analysis
```
