# Week 4 — Analysis, Progress & Blocker Report
**Target:** medirozahospital.com (NetworkWalks VAPT lab target — Batch B082)
**Intern:** Raphael Osindi — Application ID MV-CYB-2026-0092
**Date of analysis:** 2026-11-25

---

## 1. What has been done (evidence collected)

| Phase | Status | Evidence |
|---|---|---|
| OSINT / scope confirmation | Complete | `reconss1.png` (Google OSINT), WHOIS logs |
| DNS / subdomain enumeration | Complete | `reconn2.txt`, `full.txt` |
| Port & service scanning | Complete (with a target-IP error) | `192.188.201.16.*`, `webapp_*.nmap` |
| Web technology fingerprinting | Complete | whatweb / wafw00f logs in `full.txt` |
| TLS/SSL analysis | Complete | `openssl.png`, `sslscan` output |
| Web content / vuln scanning | Complete (blocked by WAF) | `nikto` output, `robots.txt`, `sitemap.xml` |
| **Gaining access** | **PARTIAL** — valid user `admin` confirmed; password not recovered | `users.png`, corrected hydra logs |
| Exploitation | Not achieved | — |
| Post-exploitation | Not reached | — |
| Reporting | Partial (recon-only PDF, no images) | `recon_full_report.pdf` |

### Key facts established
- **IP:** 199.188.201.16 — `server274-1.web-hosting.com` (US) — Namecheap shared hosting / cPanel.
- **Registrar:** Namecheap; created 2026-08-14; privacy-protected; DNSSEC unsigned.
- **Subdomains:** `ftp`, `mail`, `webmail`, `heartpatient`, `autodiscover`, `www` (CNAME).
- **Open TCP ports (domain):** 21, 25, 26, 53, 80, 110, 143, 443, 465, 587, 993, 995.
- **Web server:** LiteSpeed + `x-turbo-charged-by: LiteSpeed`; **LiteSpeed WAF** active.
- **TLS:** TLS 1.2/1.3 only, strong ciphers; cert CN `medirozahospital.com` (SSL.com, RSA-2048, valid Aug 2026–Feb 2027); **HSTS and several security headers missing**.
- **Exposed:** `/patient/`, `/staff/`, `/old/` (directory indexing, CWE-548), `/cpanel/`, `/securecontrolpanel/`, `/webmail/` (IlohaMail 0.8.10), `/mailman/listinfo`, `/sitemap.xml`.

---

## 2. Where you are stuck — root causes

### Blocker A — The whole site is now behind a LiteSpeed anti-bot JS challenge
Today every endpoint (including the homepage) returns **HTTP 200** with:
> "One moment, please..." / "Please wait while your request is being verified..."

This is a JavaScript/browser-verification interstitial. Automated tools (curl, hydra, nikto, wpscan, ffuf) cannot pass it. The aggressive scanning appears to have triggered IP reputation / rate limiting, so the challenge is now **site-wide**. *(Fresh screenshots: `evidence/shot_home.png`, `evidence/shot_patient_login.png`.)*

### Blocker B — hydra's "16 valid passwords" was a FALSE POSITIVE
The final run was:
```
hydra -l user.txt -P /usr/share/wordlists/rockyou.txt medirozahospital.com \
  https-post-form "/patient/login.php:username=^USER^&password=^PASS^:F=incorrect"
```
Why every result is wrong:
1. **`-l user.txt`** uses the literal string `user.txt` as the username — that is a filename, not a real account.
2. **Failure string `F=incorrect`** never appears in the response. The login endpoint returns the anti-bot page ("Please wait while your request is being verified..."), which does **not** contain "incorrect", so **every** attempt is counted as a success.
3. Verified directly: `curl https://medirozahospital.com/patient/login.php` → "One moment, please..." page; `grep -i incorrect` → no match.

**Conclusion: no credentials were actually found and no authenticated session was ever obtained.** *(Redacted screenshot: `evidence/users_terminal_redacted.png`.)*

### Blocker C — Wrong target IP in the full-port scans
`nmap ... -p- -oA 192.188.201.16` and `-oA webapp_full_scan` targeted **192.188.201.16** instead of **199.188.201.16** (note `192` vs `199`). The full port scan therefore only found `53/tcp Mikrotik dnsd` on an unrelated host and gave no real web-port data. The `webapp_web_ports` scan then showed 80/443 as "filtered" because it also targeted the wrong IP.

### Blocker D — Tooling / wordlist gaps
- `ffuf` failed: wordlists `backup-extenstions.txt`, `api-extenstions.txt` do not exist (also typo "extenstions").
- `dirb` URL/wordlist concatenated into one argument.
- `waybackurls`, `droopescan`, `testssl.sh` not installed; `paramspider.py` missing.
- `sublist3r` and `amass` crashed on the local Python/libpostal environment.

---

## 3. Recommended path forward (authorized lab only)

1. **Stop automated traffic for a while** and ask NetworkWalks to reset/whitelist your IP if the lab provides a reset. The JS challenge should clear once the rate-limit expires.
2. **Use a real browser** (manual or headless with JS) to pass the verification, then inspect `/patient/login.php`: form action, hidden CSRF/`__token` fields, cookies, and what a real failed login actually returns (capture the exact failure string).
3. **Retest manually** with the lab-supplied credentials or a small, documented set of lab default creds — not a full rockyou brute force.
4. **Re-run content discovery correctly** against the right IP with real wordlists:
   `ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u https://medirozahospital.com/FUZZ -rate 20 -p 0.5 -H "User-Agent: Mozilla/5.0 ..."`
5. **Check the known weak spots** first: `/old/`, `/staff/`, `/patient/` directory listings, `/webmail/` (IlohaMail 0.8.10), `/mailman/listinfo`.
6. Only after a valid session exists, move to **post-authentication enumeration** (post-exploitation) and document it.

---

## 4. Fixes applied to the original report
- Consolidated recon-only draft into a full VAPT report covering all 6 phases.
- Added **actual captured screenshots** (OSINT, TLS, hydra, live anti-bot state).
- Corrected the hydra false-positive interpretation and documented the 192/199 IP error.
- Removed/redacted all password values (submission requirement).

---

## 5. Session update — new findings & corrected credential test

### 5.1 The anti-bot challenge cleared for a real browser
Once accessed with a browser (which executes the JS), the real pages loaded. The login form is:
```html
<form method="POST" action="login.php">
  <input name="username"> <input type="password" name="password">
</form>
```
No CSRF token. Backend leaks `X-Powered-By: PHP/8.2.36`; CMS meta `Mediroza CMS 1.4.2`; `PHPSESSID` cookie set with `secure` but **no HttpOnly/SameSite**.

### 5.2 Username enumeration (CWE-204)
| Input | Response |
|---|---|
| non-existent user | `Username not found` |
| `admin` + wrong password | `Incorrect password` |
→ **`admin` is a valid account.**

### 5.3 Corrected hydra and results
```bash
hydra -l admin -P targeted_passwords.txt -t 2 -f -V -s 443 medirozahospital.com \
  https-post-form "/patient/login.php:username=^USER^&password=^PASS^:F=Incorrect password"
```
- `targeted_passwords.txt` (130 entries): **0 false positives, 0 hits**
- seclists `best1050.txt` (1,050 entries): **0 hits** (rate ≈40 attempts/min)

Method is now correct; the password is not a common one. Large lists (rockyou) are impractical here.

### 5.4 High-impact weaknesses discovered in the exposed directories
- **`/old/mediroza_db_backup_2019.sql`** (6.3 KB, unauthenticated) — `staff` + `shareholders` PII: names, roles, emails, phones, **national ID numbers, salaries**. **Critical / CWE-538.**
- **`/patient/`** directory listing — `reports/`, `download.php`, `error_log` (251 KB), `portal.php`, `logout.php`.
- **`/staff/`** listing — `login.php`.
- `download.php` enforces auth (302); path traversal blocked (403).
- `error_log` direct access → 403.

### 5.5 Status
- Gaining access: **partial** (valid user, no password).
- Exploitation / Post-exploitation: **not reached**.
- Path forward: use lab-supplied credentials, or a password derived from the leaked staff identities / targeted guess list; then verify by obtaining a session (302 → `portal.php`, report download works).

### 5.6 New evidence files
`evidence/shot_old_listing.png`, `evidence/shot_patient_listing.png`, `evidence/shot_login_real.png`, `evidence/mediroza_db_backup_2019.sql` (PII — not included in the report), `hydra_run.log`, `hydra_1050.log`.
