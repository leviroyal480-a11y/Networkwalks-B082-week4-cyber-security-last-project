# PENETRATION TESTING REPORT

## Mediroza General Hospital

### Web Application Security Assessment

**Prepared by:** Emmanuel Bafi  
**Cybersecurity Mentor:** Waqas Karim, CCIE  
**Organisation:** Networkwalks  
**Batch:** B082 | Week 4 Capstone Project  
**Target:** [https://medirozahospital.com](https://medirozahospital.com)  
**Classification:** Confidential

---

# Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope and Methodology](#2-scope-and-methodology)
   - [2.1 Scope](#21-scope)
   - [2.2 Methodology](#22-methodology)
   - [2.3 Tools Used](#23-tools-used)
   - [2.4 Files and Evidence Reviewed](#24-files-and-evidence-reviewed)
3. [Findings and Proof of Exploitation](#3-findings-and-proof-of-exploitation)
   - [3.1 Summary Table](#31-summary-table)
   - [3.2 Finding 1 — Username Enumeration](#32-finding-1--username-enumeration)
   - [3.3 Finding 2 — SQL Injection Login Bypass](#33-finding-2--sql-injection-login-bypass)
   - [3.4 Finding 3 — Confidential PDFs Accessible After Login Bypass](#34-finding-3--confidential-pdfs-accessible-after-login-bypass)
   - [3.5 Finding 4 — Weak PDF Passwords Crackable with a Wordlist](#35-finding-4--weak-pdf-passwords-crackable-with-a-wordlist)
   - [3.6 Finding 5 — Sensitive Metadata in Patient PDF Files](#36-finding-5--sensitive-metadata-in-patient-pdf-files)
   - [3.7 Finding 6 — Forgotten Backup Folder with Directory Listing Enabled](#37-finding-6--forgotten-backup-folder-with-directory-listing-enabled)
   - [3.8 Finding 7 — Confidential Staff Salaries and Shareholder Data in Plain Text](#38-finding-7--confidential-staff-salaries-and-shareholder-data-in-plain-text)
4. [Full Attack Chain Summary](#4-full-attack-chain-summary)
5. [Recommendations and Remediation](#5-recommendations-and-remediation)
   - [5.1 Fix Username Enumeration](#51-fix-username-enumeration)
   - [5.2 Fix SQL Injection](#52-fix-sql-injection)
   - [5.3 Fix PDF Access and Password Strength](#53-fix-pdf-access-and-password-strength)
   - [5.4 Strip PDF Metadata](#54-strip-pdf-metadata)
   - [5.5 Fix Directory Listing and Remove the Backup](#55-fix-directory-listing-and-remove-the-backup)
6. [Conclusion](#6-conclusion)
7. [Disclaimer](#7-disclaimer)

---

# 1. Executive Summary

I was assigned to conduct a black-box penetration test on the web infrastructure of Mediroza General Hospital at [https://medirozahospital.com](https://medirozahospital.com). The client provided written authorisation for this assessment.

The goal was to identify vulnerabilities, demonstrate their real-world impact through controlled exploitation, and provide recommendations to improve the security posture of the organisation.

During the assessment, I identified seven vulnerabilities ranging from **Medium to Critical severity**.

The most significant finding was a **SQL injection vulnerability on the patient portal login page**, which allowed authentication to be bypassed without knowing any valid credentials. This provided access to three confidential patient laboratory report PDFs.

After cracking the PDF encryption using authorised password-cracking tools, sensitive metadata was discovered inside one of the files. The metadata pointed to a forgotten database backup stored in a publicly accessible folder on the server.

The database backup contained the monthly salaries of all 30 hospital employees and the shareholding details of 10 shareholders.

The overall security posture of the target was assessed as poor. Multiple vulnerabilities could allow an unauthenticated attacker to access confidential patient data, staff financial records, and corporate ownership information with minimal effort and without specialised equipment.

> **Overall Risk Rating: CRITICAL — Immediate remediation is recommended.**

---

# 2. Scope and Methodology

## 2.1 Scope

The assessment was limited to the following target as agreed with the client.

**Target domain:** [https://medirozahospital.com](https://medirozahospital.com)

The following activities were explicitly excluded from scope:

- Social engineering
- Denial-of-service attacks
- Testing outside the agreed target domain
- Any activity not covered by the written authorisation

---

## 2.2 Methodology

I followed a structured black-box penetration testing methodology consisting of four primary phases:

### Reconnaissance

Passive information gathering was conducted using publicly available information and web-based tools.

### Vulnerability Identification

The application's behaviour was analysed to identify weaknesses in authentication, input handling, file access, and server configuration.

### Exploitation

Identified vulnerabilities were demonstrated through controlled exploitation within the authorised scope.

### Documentation

All findings, evidence, affected locations, exploitation results, and remediation recommendations were documented in this report.

---

## 2.3 Tools Used

| Tool | Purpose |
|---|---|
| `curl` | Command-line tool for sending HTTP requests and reading web server responses |
| Browser Developer Tools | Inspecting page source and login form behaviour |
| Networkwalks Hash Calculator | Extracting password hashes from PDF files |
| Networkwalks Password Cracker | Cracking PDF password hashes using wordlists |
| `qpdf` | Decrypting password-protected PDF files after authorised password recovery |
| `exiftool` | Reading hidden metadata from PDF files |
| `wget` | Downloading files from the web server |
| ChatGPT | Converting extracted SQL data into readable tables |

---

## 2.4 Files and Evidence Reviewed

The following files and evidence were identified or generated during the assessment.

| File / Resource | Type | Purpose / Evidence |
|---|---|---|
| `patient_report_1.pdf` | PDF | Confidential patient laboratory report recovered after portal access |
| `patient_report_2.pdf` | PDF | Confidential patient laboratory report recovered after portal access |
| `patient_report_3.pdf` | PDF | Confidential patient report containing metadata pointing to the database backup |
| `report3_open.pdf` | PDF | Decrypted copy of `patient_report_3.pdf` used for metadata analysis |
| `mediroza_db_backup_2019.sql` | SQL Database Backup | Forgotten database backup containing staff and shareholder records |
| `/patient/` | Web Directory | Patient portal discovered during reconnaissance |
| `/staff/` | Web Directory | Staff-related directory identified during reconnaissance |
| `/old/` | Web Directory | Publicly accessible directory containing the database backup |
| `robots.txt` | Text/Web Resource | Revealed hidden/disallowed directories during reconnaissance |
| `Mediroza_Shareholders_and_Staff.xlsx` | Excel Workbook | Readable extraction of staff and shareholder information from the SQL backup |

> **Evidence Handling Note:** The files and records referenced in this report were handled as part of an authorised educational penetration-testing exercise. Sensitive information should be redacted before public distribution of this report.

---

# 3. Findings and Proof of Exploitation

## 3.1 Summary Table

| # | Vulnerability | Location | Risk |
|---|---|---|---|
| 1 | Username enumeration on login page | `patient/login.php` | Medium |
| 2 | SQL injection login bypass | `patient/login.php` | Critical |
| 3 | Encrypted PDFs accessible after login bypass | `patient/reports/` | High |
| 4 | Weak PDF passwords crackable with a wordlist | `patient_report_*.pdf` | High |
| 5 | Sensitive metadata left in patient PDF files | `patient_report_3.pdf` | Medium |
| 6 | Forgotten backup folder with directory listing enabled | `/old/` | Critical |
| 7 | Confidential staff salaries and shareholder data in plain text | `/old/mediroza_db_backup_2019.sql` | Critical |

---

## 3.2 Finding 1 — Username Enumeration

**Risk Rating:** Medium

**Location:** `patient/login.php`

### Description

Username enumeration occurs when a login page reveals whether a username exists by displaying different error messages for a wrong username and a wrong password.

A secure login system should return the same generic error message for both cases to prevent attackers from confirming valid usernames.

### Steps Taken

I opened the patient portal login page and tested the error messages by entering different inputs.

First, I entered a username that I assumed would think of first, but.

```text
username: bob
password: test123
```
## Response

```text
### Username not found
```
I then tried a common default username.

```text
**Username:** `admin`  
**Password:** `test123`
```

### Response
```text
> Incorrect password
```

The two different responses confirmed that `admin` was a valid account on the system.

This reduced the authentication attack surface because the username was now known and only the password remained to be discovered or bypassed.

### Evidence screenshot 


---

## 3.3 Finding 2 — SQL Injection Login Bypass

**Risk Rating:** Critical

**Location:** `patient/login.php`

### Description

SQL injection occurs when an application places user-controlled input directly into a database query without properly separating SQL code from user input.

In this case, the username field was vulnerable to SQL injection. Controlled testing demonstrated that the authentication query could be manipulated so that the password validation was bypassed.

### Steps Taken

I tested the username field for SQL injection by entering a single quote.

```text
**Username:** `admin'`  
**Password:** `test123`
```

### Response

```text
Warning: mysqli_query(): You have an error in your SQL syntax;
check the manual that corresponds to your MySQL server version
for the right syntax to use near ''' at line 1
