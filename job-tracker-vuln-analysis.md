---
title: "Security Analysis of a Python Job Application Tracker"
date: 2026-04-16
layout: default
description: "Applying real-world vulnerability analysis techniques to a personal Python project — a Gmail-integrated job application tracker built with SQLAlchemy, Dash, and the Google API."
---

# Security Analysis of a Python Job Application Tracker

## Overview

One of the best ways to sharpen your vulnerability analysis skills is to turn the lens on your own projects. I recently built a personal job application tracker in Python that automatically monitors my Gmail inbox, parses application status emails, stores records in a MySQL database, and visualizes everything in a Dash dashboard. It's a practical tool — but it's also a useful target for a real security assessment.

This post walks through applying the same vulnerability analysis methodology I use professionally (OWASP Top 10, threat modeling, CVSS-informed prioritization) to this project, covering real findings and concrete remediations.

---

## Project Architecture

Before analyzing vulnerabilities, understanding the attack surface is essential.

**Stack:**
- **Data layer:** MySQL via SQLAlchemy + PyMySQL, seeded from CSV or live Gmail polling
- **Email integration:** Gmail API (OAuth 2.0) via `google-api-python-client`
- **Analysis:** Pandas, Scikit-learn, Jupyter Notebooks
- **Visualization:** Plotly Dash dashboard with Bootstrap components
- **Config:** `.env` file loaded via `python-dotenv`

**Data flow:**
```
Gmail Inbox → email_monitor.py (OAuth) → email_parser.py (regex) → db.py (SQLAlchemy) → MySQL
CSV files   → ingest.py (pandas)       → db.py                   → MySQL
                                                                     ↓
                                                          dashboards/app.py (Dash)
```

**Sensitive assets at a glance:**
| Asset | Location | Sensitivity |
|---|---|---|
| DB credentials | `.env` | High |
| Gmail OAuth credentials | `config/gmail_credentials.json` | Critical |
| Gmail OAuth token | `config/gmail_token.json` | Critical |
| Application data (PII) | MySQL `applications` table | Medium |
| Raw CSV imports | `data/raw/` | Medium |

---

## Threat Model

**Threat actors considered:**
- Attacker with read access to the filesystem (e.g., misconfigured file share, stolen laptop)
- Attacker with network access to the MySQL host (`192.168.0.50`)
- Attacker who obtains the repository (e.g., accidental public push)

**Primary concerns:**
1. Credential / token exposure
2. Unauthorized database access
3. Vulnerable dependencies
4. Dash dashboard without authentication
5. Unsafe file ingestion

---

## Finding 1 — Credential Exposure via Accidental Git Commit

**OWASP:** A02:2021 – Cryptographic Failures  
**Severity:** Critical (CVSS 9.1)

The most dangerous finding has nothing to do with runtime behavior — it's the risk of committing secrets to version control. The project stores three sensitive files that would be catastrophic if pushed to a public repository:

```
config/gmail_credentials.json   ← Google OAuth client ID + secret
config/gmail_token.json         ← Live access + refresh token for your Gmail account
.env                            ← DB_USER, DB_PASSWORD, DB_HOST
```

`settings.py` constructs the database URL directly from environment variables:

```python
DATABASE_URL = (
    f"mysql+pymysql://{DB_USER}:{DB_PASSWORD}"
    f"@{DB_HOST}:{DB_PORT}/{DB_NAME}?charset=utf8mb4"
)
```

If `DB_HOST` defaults to `192.168.0.50` (as it does in the current code) and that IP is reachable, an attacker with the `.env` file has full database access. If `gmail_token.json` leaks, they have read access to your entire Gmail inbox — including every job offer, salary negotiation, and personal email.

**Remediations:**
1. Add all three to `.gitignore` immediately:
   ```
   .env
   config/gmail_credentials.json
   config/gmail_token.json
   data/raw/
   data/processed/
   ```
2. Rotate the Google OAuth credentials in Google Cloud Console if there's any chance they were ever committed.
3. Remove the hardcoded `DB_HOST` default — fail loudly if the variable is absent rather than defaulting to a real IP:
   ```python
   DB_HOST = os.getenv('DB_HOST')
   if not DB_HOST:
       raise EnvironmentError("DB_HOST is not set")
   ```
4. Consider using a secrets manager (Vault, AWS Secrets Manager, or even `keyring`) rather than plaintext `.env` files for production use.

---

## Finding 2 — Dash Dashboard Has No Authentication

**OWASP:** A01:2021 – Broken Access Control  
**Severity:** High (CVSS 7.5)

The Dash application (`dashboards/app.py`) runs as a standard Flask/Dash server. By default, Dash exposes the app on `0.0.0.0` with no authentication layer. Anyone on the local network who knows the port can view all job application data, including companies, roles, salary stages, and notes.

On a home network this may seem low risk, but consider:
- The MySQL host is already at a static LAN IP (`192.168.0.50`) — suggesting it's a reachable home server
- A guest device or a compromised device on the same network gets full dashboard access
- If the machine is ever port-forwarded or on a VPN, exposure widens significantly

**Remediations:**
1. Bind the Dash server to `127.0.0.1` only for local use:
   ```python
   app.run(host="127.0.0.1", port=8050, debug=False)
   ```
2. For network access, add HTTP Basic Auth using `dash-auth` or Flask's `flask-login`:
   ```python
   from dash_auth import BasicAuth
   VALID_USERNAME_PASSWORD_PAIRS = {'admin': os.getenv('DASH_PASSWORD')}
   auth = BasicAuth(app, VALID_USERNAME_PASSWORD_PAIRS)
   ```
3. Add `DASH_PASSWORD` to `.env` rather than hardcoding credentials.

---

## Finding 3 — Unpinned Dependencies with Known Vulnerability Surface

**OWASP:** A06:2021 – Vulnerable and Outdated Components  
**Severity:** Medium (CVSS 6.5)

`requirements.txt` lists every dependency without version pins:

```
pandas
sqlalchemy
pymysql
plotly
dash
scikit-learn
google-api-python-client
...
```

This means `pip install -r requirements.txt` always installs the latest version — which sounds safe but introduces real risks:
- A dependency may release a breaking or malicious version (supply chain attack)
- You cannot reproduce the exact environment used when the project was last known-good
- `pip audit` cannot check for CVEs without pinned versions to look up

In early 2025, the `requests` package ecosystem saw several typosquatting incidents. The `dash` and `flask` ecosystem has had multiple CVEs related to CORS misconfigurations and debug mode exposure.

**Remediations:**
1. Pin all dependencies after a clean install:
   ```bash
   pip install -r requirements.txt
   pip freeze > requirements.txt
   ```
2. Run `pip audit` or integrate `safety check` into your workflow:
   ```bash
   pip install pip-audit
   pip-audit -r requirements.txt
   ```
3. Use a virtual environment with a lockfile (Poetry or `pip-compile` from `pip-tools`) for reproducible installs.

---

## Finding 4 — CSV Ingestion Without Input Validation

**OWASP:** A08:2021 – Software and Data Integrity Failures  
**Severity:** Medium (CVSS 5.4)

`ingest.py` reads arbitrary CSV files from the filesystem and inserts their contents into MySQL. While the code does truncate string fields (e.g., `company[:100]`) and validates the `status` field against a whitelist, there are gaps:

```python
def ingest_csv(filepath: str) -> int:
    df = pd.read_csv(filepath)           # No file type verification
    df.columns = df.columns.str.strip()  # Column names from file are trusted
    ...
    execute("""
        INSERT IGNORE INTO applications (date_applied, company, role, status, notes)
        VALUES (:date_applied, :company, :role, :status, :notes)
    """, { ... })                        # Parameterized — SQL injection not possible
```

The SQL itself is safe — parameterized queries via SQLAlchemy's `text()` with bound parameters prevent SQL injection. However:
- A crafted CSV could supply a `notes` field with 500 characters of XSS payload that renders in the Dash dashboard
- A malicious CSV could supply a `date_applied` value that causes unexpected behavior in date parsing
- There is no verification that the file is actually a CSV (e.g., a zip bomb or a file with a `.csv` extension containing binary data)

**Remediations:**
1. Sanitize the `notes` field — strip HTML tags before storage:
   ```python
   import html
   notes = html.escape(str(row.get('notes', '')))[:500]
   ```
2. Validate date format strictly:
   ```python
   from datetime import datetime
   try:
       date_val = datetime.strptime(str(row['date_applied']), '%Y-%m-%d').date()
   except ValueError:
       continue  # Skip malformed rows
   ```
3. Add a file size check before reading with pandas — prevent processing unexpectedly large files.

---

## Finding 5 — OAuth Token Stored as World-Readable File (Partially Mitigated)

**OWASP:** A02:2021 – Cryptographic Failures  
**Severity:** Medium (CVSS 5.9)

`email_monitor.py` saves the Gmail OAuth token after the first auth flow:

```python
TOKEN_FILE.write_text(creds.to_json())
TOKEN_FILE.chmod(0o600)     # ← Good: owner-only read/write
log.info("Token saved to %s", TOKEN_FILE)
```

The `chmod(0o600)` is a positive security control — it restricts the token file to the owner only. However:
- The token is stored in plaintext JSON — any process running as the same user can read it
- If the refresh token leaks, it can generate new access tokens indefinitely until revoked
- The token file path (`config/gmail_token.json`) is inside the project directory, adjacent to code that may be shared or synced

**Remediations:**
1. Store the token outside the project directory (e.g., `~/.config/job_tracker/gmail_token.json`)
2. Consider encrypting the token at rest using the system keychain:
   ```python
   import keyring
   keyring.set_password("job_tracker", "gmail_token", creds.to_json())
   ```
3. Implement token revocation on application exit for sensitive workflows.

---

## What Was Done Well

Not every finding is a problem — good security practices deserve recognition:

| Practice | Code Location | Why It Matters |
|---|---|---|
| Parameterized SQL queries | `db.py`, `ingest.py` | Prevents SQL injection completely |
| Status field allowlist | `ingest.py` | Prevents unexpected values from entering the DB |
| OAuth `gmail.readonly` scope only | `email_monitor.py` | Limits blast radius if token is compromised |
| Token file `chmod(0o600)` | `email_monitor.py` | Restricts filesystem access to owner |
| `.env` for credentials | `config/settings.py` | Avoids hardcoding secrets in source |
| Field length truncation | `ingest.py` | Prevents oversized inserts |

---

## Prioritized Remediation Plan

| Priority | Finding | Effort | Impact |
|---|---|---|---|
| P0 | Add secrets to `.gitignore`, rotate if committed | 5 min | Critical |
| P1 | Bind Dash to `127.0.0.1` or add basic auth | 15 min | High |
| P2 | Pin dependencies + add `pip-audit` | 20 min | Medium |
| P3 | Sanitize `notes` field (HTML escape) | 10 min | Medium |
| P4 | Move token file outside project dir | 10 min | Medium |
| P5 | Add hard failure on missing env vars | 5 min | Low |

---

## Lessons Learned

Personal projects are easy to treat as "low stakes" from a security perspective — but they often have the most sensitive access of anything you run. This tracker has OAuth access to a Gmail inbox containing salary negotiations, offer letters, and background check confirmations. That warrants the same rigor as any professional system.

The key takeaways from this analysis:

1. **Your `.gitignore` is a security control.** Treat it as one.
2. **Parameterized queries matter** — even in personal projects, they're the right habit.
3. **Principle of least privilege** applies to OAuth scopes, not just IAM roles.
4. **Dependency pinning is both a stability and a security practice.**
5. **Local network ≠ trusted network** — bind services explicitly.

The same methodology used here — threat modeling, OWASP mapping, CVSS-informed prioritization, and concrete remediation steps — applies directly to enterprise vulnerability programs. The scale changes; the framework doesn't.

---

*Project source: `/mnt/lee/Code_Projects/ActiveCode/Python/job_application_tracker/`*
