---
title: "Burp Suite for Web Application Penetration Testing"
date: 2026-04-15
layout: default
description: "A practical guide to using Burp Suite Community and Pro for web application security testing, OWASP Top 10 vulnerability discovery, and manual analysis."
---

# Burp Suite for Web Application Penetration Testing

## Overview

Burp Suite is the industry-standard tool for web application penetration testing. Where Nmap maps the network layer and Tenable identifies known CVEs, Burp Suite goes deeper into the application layer — intercepting HTTP/S traffic, manipulating requests, fuzzing parameters, and identifying vulnerabilities that automated scanners miss. This guide walks through how I use Burp Suite to find and validate web application vulnerabilities, with direct ties to the OWASP Top 10 categories covered in my earlier post.

---

## Step 1: Environment Setup & Proxy Configuration

Before testing, configure Burp Suite as an intercepting proxy between your browser and the target:

- Burp Suite listens on `127.0.0.1:8080` by default
- Configure your browser to route traffic through this proxy
- Install the Burp CA certificate to intercept HTTPS traffic without SSL errors

**Setup Steps:**
1. Launch Burp Suite and navigate to **Proxy > Options**
2. Confirm the proxy listener is running on `127.0.0.1:8080`
3. In your browser, set the HTTP/HTTPS proxy to `127.0.0.1:8080`
4. Navigate to `http://burpsuite` (or `http://burp`) in your browser
5. Download and install the Burp CA certificate in your browser's certificate store
6. Enable **Intercept** under **Proxy > Intercept** to begin capturing traffic

**Recommended Browser Setup:**
- Use Firefox with the FoxyProxy extension for easy proxy toggling
- Create a dedicated browser profile for testing to avoid contaminating personal browsing

---

## Step 2: Traffic Interception & Manual Analysis

With the proxy running, browse the target application to build a complete traffic map:

- Walk through every page and feature of the application with intercept off
- Burp automatically populates the **Target > Site Map** as you browse
- Enable intercept selectively to inspect and modify specific requests

**What to Examine in Each Request:**
- Authentication tokens and session cookies — are they predictable or improperly scoped?
- Hidden form fields and parameters — often forgotten and under-validated
- HTTP methods — are `PUT`, `DELETE`, and `PATCH` accepted where they shouldn't be?
- Request headers — `X-Forwarded-For`, `Referer`, `Origin` can bypass access controls
- Error responses — verbose errors often leak server software versions and stack traces

**Key Areas in Burp:**
- **HTTP History** (`Proxy > HTTP History`) — full log of all intercepted requests
- **Site Map** (`Target > Site Map`) — visual tree of discovered endpoints
- **Scope** (`Target > Scope`) — restrict Burp activity to your authorized target only

---

## Step 3: Scanning with Burp Scanner (Pro)

Burp Suite Pro includes an automated crawler and vulnerability scanner:

- Right-click any target in the Site Map and select **Scan** to launch
- Configure scan type: **Crawl and Audit**, **Crawl only**, or **Audit selected items**
- Review scanner findings under **Dashboard > Issue activity**

**Burp Scanner Detects:**
- SQL injection (A03:2021)
- Cross-site scripting — reflected, stored, and DOM-based (A03:2021)
- Security misconfigurations — CORS, clickjacking headers, TLS issues (A05:2021)
- Sensitive data exposure — credentials in responses, directory listings (A02:2021)
- SSRF — internal service requests triggered by user input (A10:2021)

> **Note:** Burp Community edition does not include the automated scanner. Manual testing techniques in the following steps apply to both Community and Pro.

---

## Step 4: Manual Vulnerability Testing

Automated scanners miss business logic flaws, authorization issues, and chained vulnerabilities. Manual testing covers what automation cannot:

### SQL Injection (A03:2021)

Inject SQL syntax into input fields, URL parameters, headers, and cookies:

```
# Basic detection payloads
'
''
' OR '1'='1
' OR 1=1--
' UNION SELECT NULL--

# Send to Repeater, modify, and observe response differences
```

- Use **Repeater** to craft and replay modified requests
- Look for database error messages, changed response sizes, or boolean behavior differences
- Use **sqlmap** to automate exploitation once injection is confirmed

### Cross-Site Scripting (A03:2021)

Test all user-controlled input that appears in HTML responses:

```
# Basic XSS payloads
<script>alert(1)</script>
"><script>alert(1)</script>
<img src=x onerror=alert(1)>
javascript:alert(1)
```

- Use Burp's **DOM Invader** (built-in browser tool) to detect DOM-based XSS
- Check reflected input in headers, error messages, and search results
- Test stored XSS in comment fields, profile data, and any persisted user input

### Broken Access Control (A01:2021)

Test whether authorization is properly enforced:

- Log in as a low-privilege user and capture requests to sensitive endpoints
- Use **Burp Comparer** to diff responses between privilege levels
- Modify user IDs, account numbers, and object references in requests (IDOR testing)
- Test horizontal and vertical privilege escalation separately

```
# Example: Change the user ID in a request and see if another user's data is returned
GET /api/user/1001/profile → change to /api/user/1002/profile
```

- Install the **Autorize** Burp extension for automated IDOR detection across user roles

### Authentication Failures (A07:2021)

Test login mechanisms and session management:

- Send login requests to **Intruder** for credential stuffing or brute-force testing
- Check if account lockout is enforced after failed attempts
- Inspect session token entropy — use **Sequencer** to analyze randomness
- Test password reset flows for token predictability and expiration

```
# Intruder attack types for credential testing
Sniper — single payload position (username or password wordlist)
Cluster bomb — multiple payload sets (username + password combo lists)
```

### SSRF (A10:2021)

Look for parameters that accept URLs or IP addresses as input:

```
# Test for internal service access
?url=http://127.0.0.1/admin
?url=http://169.254.169.254/latest/meta-data/   # AWS metadata endpoint
?redirect=http://internal-server.local/
```

- Use Burp Collaborator (Pro) to detect out-of-band SSRF where responses are not directly visible
- Test all URL parameters, `Referer` headers, and XML/JSON fields containing hostnames

---

## Step 5: Using Burp Intruder for Fuzzing

Intruder automates payload injection across parameterized requests:

1. Capture a request in Proxy and send it to **Intruder** (`Ctrl+I`)
2. Highlight payload positions with `§` markers
3. Select attack type and load a payload wordlist
4. Launch attack and analyze responses by status code, length, and content

**Useful Intruder Payloads:**
- Directory and file wordlists (from SecLists) — discover hidden endpoints
- Common password lists — credential brute-force against login forms
- Special characters — injection and fuzzing payloads
- IDOR numeric sequences — test object reference enumeration

> **Note:** Burp Community throttles Intruder speed. Use **ffuf** or **feroxbuster** for large-scale fuzzing and reserve Intruder for targeted manual attacks.

---

## Step 6: Burp Repeater for Manual Exploitation

Repeater is the most-used Burp tool for hands-on vulnerability validation:

- Send any intercepted request to Repeater (`Ctrl+R`)
- Modify headers, parameters, cookies, and body content freely
- Compare responses side-by-side to understand application behavior
- Chain multiple requests to test multi-step attack flows

**Common Repeater Workflows:**
- Confirm SQL injection by observing database errors or data leakage
- Test authentication bypass by manipulating session tokens
- Exploit broken object-level authorization by swapping user identifiers
- Reproduce and document confirmed vulnerabilities for reporting

---

## Step 7: Reporting Findings

Document each finding with enough detail to reproduce and remediate:

- Capture the exact request and response that demonstrates the vulnerability
- Use Burp's **Logger** or right-click any request to copy as `curl` for reproduction steps
- Map findings to OWASP Top 10 categories and assign risk ratings
- Include remediation guidance alongside each finding

**Finding Documentation Template:**

| Field | Content |
|-------|---------|
| Title | Reflected XSS in Search Parameter |
| OWASP Category | A03:2021 – Injection |
| Severity | High |
| Endpoint | `GET /search?q=<payload>` |
| Proof of Concept | Request/response screenshots |
| Impact | Session hijacking, credential theft |
| Remediation | Input sanitization, Content-Security-Policy header |

---

## Integrating Burp Suite Into the Broader Workflow

Burp Suite is the web application layer of your full testing pipeline:

- **Nmap** identifies web servers on ports 80, 443, 8080, 8443 — these become your Burp targets
- **Tenable** may flag outdated web server versions — use Burp to validate whether those versions are actually exploitable via the application layer
- **Metasploit** can take over after Burp confirms a web vulnerability — use `auxiliary/scanner/http/` modules to extend coverage or deliver payloads post-exploitation
- **OWASP Top 10** provides the classification framework — map every Burp finding to its OWASP category when reporting

---

## Tools & Resources

- **[Burp Suite Community / Pro](https://portswigger.net/burp)** — Download and documentation
- **[PortSwigger Web Security Academy](https://portswigger.net/web-security)** — Free hands-on labs for every OWASP category, built by the Burp Suite team
- **[OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)** — Comprehensive manual testing methodology
- **[Autorize Extension](https://github.com/PortSwigger/autorize)** — Automated authorization testing across user roles
- **[SecLists](https://github.com/danielmiessler/SecLists)** — Payload and wordlist collection for Intruder and fuzzing
- **[DOM Invader](https://portswigger.net/burp/documentation/desktop/tools/dom-invader)** — Built-in DOM XSS detection tool

---

## Important Ethical Considerations

- Only test web applications you own or have explicit written permission to assess
- Automated scanning and fuzzing can cause unintended data modification — confirm scope and acceptable test intensity with the client before running
- Avoid testing in production environments when a staging or dev environment is available
- Credential brute-forcing against live login forms can lock out legitimate users — coordinate timing with the client

---

## Final Thoughts

Burp Suite bridges the gap between network-level vulnerability data and real application risk. Tenable will tell you a web server is running an outdated framework — Burp Suite tells you whether that framework is actually exploitable through the application, and where the business logic breaks down in ways no automated scanner would catch. Combined with Nmap for reconnaissance and Metasploit for post-exploitation validation, Burp Suite completes the full attack chain assessment for any web-facing target.
