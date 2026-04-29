# CVE-2022-22965 — Spring4Shell RCE Writeup

**Platform:** TryHackMe  
**Category:** Web Application / Remote Code Execution  
**Difficulty:** Medium  
**CVE:** CVE-2022-22965  
**CVSS Score:** 9.8 (Critical)  

---

## Vulnerability Overview

Spring4Shell is a critical Remote Code Execution (RCE) vulnerability discovered in April 2022 within the **Spring Framework**. It affects Spring MVC and Spring WebFlux applications running on **JDK 9 or later**.

The root cause lies in Java's `ClassLoader` mechanism — an attacker can abuse Spring's data binding feature to execute arbitrary code on the server **without any authentication**.

---

## Affected Versions

- Spring Framework 5.3.0 – 5.3.17
- Spring Framework 5.2.0 – 5.2.19
- JDK version 9 and above
- Apache Tomcat as the servlet container

---

## Reconnaissance

Started with a basic Nmap scan to identify open ports and running services:

```bash
nmap -sV -sC -p- <target-ip>
```

Identified an HTTP service hosting a Spring-based web application.

Opened the application in a browser and performed technology fingerprinting:
- Confirmed Spring Framework via HTTP response headers
- Confirmed JDK 9+ environment

---

## Exploitation

CVE-2022-22965 abuses Spring's data binding feature to access the `ClassLoader` property through a crafted HTTP request. This allows an attacker to modify Tomcat's logging configuration and write a malicious JSP webshell to disk.

Ran the provided Python exploit script:

```bash
python3 exploit.py --url http://<target-ip>:8080/
```

**What the exploit did:**
- Abused `class.classLoader` property via Spring data binding
- Modified Tomcat's `AccessLogValve` configuration
- Wrote a JSP webshell to `/tomcatwar.jsp`

Accessed the webshell to verify command execution:

```
http://<target-ip>:8080/tomcatwar.jsp?pwd=j&cmd=id
```

**Output:**
```
uid=0(root) gid=0(root) groups=0(root)
```

Root access confirmed.

---

## Exploitation Chain

```
HTTP Request
    │
    ▼
Spring Data Binding
    │
    ▼
ClassLoader Property Access
    │
    ▼
Tomcat AccessLogValve Manipulation
    │
    ▼
JSP Webshell Written to Disk
    │
    ▼
Remote Code Execution as Root
```

---

## Real-World Impact

If exploited against a production server, an attacker could:

- Achieve complete server takeover
- Exfiltrate sensitive data (credentials, PII, API keys)
- Plant a persistent backdoor
- Perform lateral movement into internal networks

---

## Remediation

| Action | Details |
|--------|---------|
| Upgrade Spring Framework | Patch to 5.3.18+ or 5.2.20+ |
| Downgrade JDK | Use JDK 8 as a temporary mitigation |
| WAF Rules | Block requests containing `class.classLoader` patterns |
| Input Validation | Restrict Spring data binding to whitelisted fields only |

---

## Key Takeaways

- Framework-level vulnerabilities can be catastrophic — a single unpatched dependency can lead to full system compromise.
- Default Spring configurations allow `ClassLoader` access, which should always be restricted in production.
- CVSS 9.8 means immediate patching — no exceptions.
- Always verify the framework version and JDK version during the reconnaissance phase of a web application pentest.

---

*Written by Muhammad Muzammil Khan — Penetration Tester*  
*TryHackMe Profile: https://tryhackme.com/r/p/mmuzammil.khan*
