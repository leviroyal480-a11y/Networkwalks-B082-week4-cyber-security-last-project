# PENETRATION TESTING REPORT

## Mediroza General Hospital
### Web Application Security Assessment

**Prepared by:** Monday Royal Levi 
**Cybersecurity Mentor:** Waqas Karim, CCIE  
**Organisation:** Networkwalks  
**Batch:** B082 | Week 4
**Target:** `https://medirozahospital.com`  
**Classification:** Confidential

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Scope and Methodology](#2-scope-and-methodology)
  - [2.1 Scope](#21-scope)
  - [2.2 Methodology](#22-methodology)
  - [2.3 Tools Used](#23-tools-used)
- [3. Findings and Proof of Exploitation](#3-findings-and-proof-of-exploitation)
  - [3.1 Summary Table](#31-summary-table)
  - [3.2 Finding 1 — Username Enumeration](#32-finding-1--username-enumeration)
  - [3.3 Finding 2 — SQL Injection Login Bypass](#33-finding-2--sql-injection-login-bypass)
  - [3.4 Finding 3 — Confidential PDFs](#34-finding-3--confidential-pdfs-accessible-after-login-bypass)
  - [3.5 Finding 4 — Weak PDF Passwords](#35-finding-4--weak-pdf-passwords-crackable-with-a-wordlist)
  - [3.6 Finding 5 — Sensitive Metadata](#36-finding-5--sensitive-metadata-in-patient-pdf-files)
  - [3.7 Finding 6 — Forgotten Backup Folder](#37-finding-6--forgotten-backup-folder-with-directory-listing-enabled)
  - [3.8 Finding 7 — Confidential Staff and Shareholder Data](#38-finding-7--confidential-staff-salaries-and-shareholder-data-in-plain-text)
- [4. Full Attack Chain Summary](#4-full-attack-chain-summary)
- [5. Recommendations and Remediation](#5-recommendations-and-remediation)
  - [5.1 Fix Username Enumeration](#51-fix-username-enumeration)
  - [5.2 Fix SQL Injection](#52-fix-sql-injection)
  - [5.3 Fix PDF Access and Password Strength](#53-fix-pdf-access-and-password-strength)
  - [5.4 Strip PDF Metadata](#54-strip-pdf-metadata)
  - [5.5 Fix Directory Listing and Remove the Backup](#55-fix-directory-listing-and-remove-the-backup)
- [6. Conclusion](#6-conclusion)
- [7. Disclaimer](#7-disclaimer)

---

# 1. Executive Summary

I was assigned to conduct a black-box penetration test on the web infrastructure of Mediroza General Hospital at `https://medirozahospital.com`.

The client provided written authorisation for this assessment. The goal was to identify vulnerabilities, demonstrate their real-world impact through controlled exploitation, and provide recommendations to improve the security posture of the organisation.

During the assessment, I identified seven vulnerabilities ranging from **Medium to Critical severity**.

The most significant finding was a **SQL injection vulnerability** on the patient portal login page, which allowed authentication to be bypassed without knowing valid credentials.

This provided access to confidential patient laboratory report PDFs. Further analysis identified weak PDF passwords and sensitive metadata that revealed the location of an old database backup.

The publicly accessible backup contained sensitive staff and shareholder information.

### Overall Risk Rating

> **CRITICAL — Immediate remediation is recommended.**

---

# 2. Scope and Methodology

## 2.1 Scope

The assessment was limited to the following target as agreed with the client.

**Target domain:**

```text
https://medirozahospital.com
```

---

## 2.2 Methodology

The assessment followed a structured black-box penetration testing methodology consisting of four phases:

### 1. Reconnaissance

Passive information gathering using publicly available information and web-based tools.

### 2. Vulnerability Identification

Analysing application behaviour to identify weaknesses in authentication, input handling, file access, and server configuration.

### 3. Controlled Exploitation

Demonstrating the real-world impact of identified vulnerabilities through controlled exploitation.

### 4. Documentation

Recording findings, evidence, impact, and remediation recommendations.

---

## 2.3 Tools Used

| Tool | Purpose |
|---|---|
| `curl` | Sending HTTP requests and analysing web server responses |
| Browser Developer Tools | Inspecting page source and login form behaviour |
| Networkwalks Hash Calculator | Extracting password hashes from PDF files |
| Networkwalks Password Cracker | Testing PDF password hashes against wordlists |
| `qpdf` | Decrypting password-protected PDF files after authorised password recovery |
| `exiftool` | Analysing PDF metadata |
| ChatGPT | Converting raw SQL data into readable tables |

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
| 6 | Forgotten backup folder with directory listing enabled | `old/` | Critical |
| 7 | Confidential staff salaries and shareholder data in plain text | `old/mediroza_db_backup_2019.sql` | Critical |

---

## 3.2 Finding 1 — Username Enumeration

**Risk Rating:** Medium

**Location:** `patient/login.php`

### Description

Username enumeration occurs when a login page reveals whether a username exists by displaying different error messages for an invalid username and an incorrect password.

A secure authentication system should provide a consistent error message for failed authentication attempts.

### Steps Taken

The patient portal login page was tested with different username and password combinations.

**Test 1:**

```text
Username: mediroza 
Password: 1234567
```

## Response

### Username not found

I then tried a common default username and password because I tried the default username and my previous password it said password not correct.

```text
**Username:** `admin`  
**Password:** `password`
```

### Response

> Incorrect password

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

**Username:** `admin'--`  
**Password:** `password`

### Response

```text
Warning: mysqli_query(): You have an error in your SQL syntax;
check the manual that corresponds to your MySQL server version
for the right syntax to use near ''' at line 1
```

The database error confirmed that the input was reaching the SQL query without adequate parameterisation.

The application was effectively constructing a query similar to:

```sql
SELECT * FROM users WHERE username='admin'' AND password='password'
```

The additional quote altered the SQL syntax and produced the database error.

Controlled testing then demonstrated that the authentication logic could be bypassed using a comment-based SQL injection payload.

**Username:** `admin' --`  
**Password:** `password`

The resulting query structure effectively reduced the authentication condition to the username check:

```sql
SELECT * FROM users WHERE username='admin'
```

The password condition was therefore ignored by the database parser, resulting in unauthorised access to the portal.

### Impact

Successful exploitation allowed authentication to be bypassed without knowing the legitimate account password.

This finding was the starting point for the subsequent attack chain.

### Evidence Screenshot

---

## 3.4 Finding 3 — Confidential PDFs Accessible After Login Bypass

**Risk Rating:** High  
**Location:** `patient/reports/`

### Description

After gaining unauthorised access to the patient portal through the SQL injection vulnerability, three patient laboratory report PDFs were available for download.

These documents contained confidential medical information and should only have been accessible to authorised users with a legitimate need to view them.

### Steps Taken

After successfully demonstrating the authentication bypass, I was directed to the patient portal.

The portal listed three downloadable PDF files:

- `patient_report_1.pdf`
- `patient_report_2.pdf`
- `patient_report_3.pdf`

The files were downloaded for authorised assessment and analysis.

### Evidence Screenshot

### Impact

An attacker capable of bypassing authentication could access confidential patient documents.

---

## 3.5 Finding 4 — Weak PDF Passwords Crackable with a Wordlist

**Risk Rating:** High  
**Location:** `patient_report_*.pdf`

### Description

All three PDFs were password protected. However, the passwords were weak and appeared in commonly available password wordlists.

This made the document encryption vulnerable to automated password-guessing attacks.

Using weak passwords for sensitive medical documents does not provide meaningful protection against an attacker who obtains the encrypted files.

### Steps Taken

I used the Networkwalks Hash Calculator to extract a crackable hash from each PDF.

The extracted hashes were then tested using the Networkwalks Password Cracker.

Reports 1 and 2 were successfully recovered using the built-in 100-word default wordlist.

### Results

| Report | Result |
|---|---|
| `patient_report_1.pdf` | `123456` |
| `patient_report_2.pdf` | `password` |
| `patient_report_3.pdf` | Not cracked using the built-in list |

Report 3 did not crack using the built-in list.

A larger authorised password wordlist(JTR default) was then used for the assessment.

```text
patient_report_3.pdf → !@#$%^&
```

The recovered passwords were then used to open the documents and verify their contents.

### Evidence Screenshot

### Impact

An attacker who obtained the encrypted documents could recover their passwords using common wordlists and gain access to confidential medical information.

---

## 3.6 Finding 5 — Sensitive Metadata in Patient PDF Files

**Risk Rating:** Medium  
**Location:** `patient_report_3.pdf`

### Description

PDF files can contain metadata such as the author, creation date, software information, and custom comments.

Although this information may not be visible when reading the document normally, it can be extracted using tools such as `exiftool`.

In this case, `patient_report_3.pdf` contained an internal staff comment that pointed directly to a database backup stored on the server.

### Steps Taken

I first decrypted Report 3 using `qpdf` so that its metadata could be fully analysed.

```bash
qpdf --password='!@#$%^&' --decrypt patient_report_3.pdf report3_open.pdf
```

I then ran `exiftool` against the unlocked file.

```bash
exiftool report3_open.pdf
```

### Key Fields

```text
Author   : j.malik
Comments : DB backup moved to /old/ before site migration, do not delete
```

The metadata revealed the location of a database backup on the server.

The author identifier `j.malik` was later correlated with the staff records recovered from the database backup.

### Evidence Screenshot

### Impact

The metadata disclosure provided an attacker with an internal operational clue that directly assisted in locating sensitive server-side data.

---

## 3.7 Finding 6 — Forgotten Backup Folder with Directory Listing Enabled

**Risk Rating:** Critical  
**Location:** `/old/`

### Description

Directory listing is a web server misconfiguration that allows visitors to view the contents of a directory when no index page exists.

The `/old/` folder had directory listing enabled and exposed a database backup file to unauthenticated visitors.

The folder had already been identified during reconnaissance because it appeared as a disallowed path in `robots.txt`.

The metadata discovered in Finding 5 then confirmed that `/old/` was the location of the database backup.

### Steps Taken

During reconnaissance, I noted `/old/` in the site's `robots.txt` file.

After discovering the metadata clue from Finding 5, I accessed the directory:

```text
https://medirozahospital.com/old/
```

The directory listing displayed the following database backup:

```text
mediroza_db_backup_2019.sql
```

The file was then downloaded for authorised analysis.

```


### Evidence 
screenshot

### Impact

An unauthenticated visitor could directly discover and download a database backup containing sensitive organisational information.

---

## 3.8 Finding 7 — Confidential Staff Salaries and Shareholder Data in Plain Text

**Risk Rating:** Critical  
**Location:** `/old/mediroza_db_backup_2019.sql`

### Description

The database backup contained highly sensitive information in plain text.

The staff records included:

- Employee names
- Job titles
- Departments
- Email addresses
- Phone numbers
- National identification information
- Monthly salaries

The backup also contained shareholder information, including:

- Shareholder names
- Shareholding percentages
- Share classes

The database backup was accessible without authentication through the publicly exposed `/old/` directory.

### Steps Taken

I opened the SQL backup in a text editor and reviewed the relevant `INSERT INTO` statements.

The staff records were extracted and converted into a readable table.

The shareholder records were also extracted and converted into a readable table.

### Extracted Staff Records

| Name | Job Title | Department | Monthly Salary (ZAR) |
|---|---|---|---|
| Dr. Rajesh Naidoo | Chief Pathologist | Diagnostics Lab | 138,000 |
| Sarah Botha | Chief Financial Officer | Finance | 152,000 |
| Dr. Johan van der Merwe | Medical Director | Management | 160,000 |
| Dr. Anita Naicker | Consultant Cardiologist | Cardiology | 132,000 |
| Dr. Ahmed Kara | Consultant Physician | Internal Medicine | 128,000 |
| Dr. Yusuf Cassim | Senior Registrar | Emergency & Trauma | 74,000 |
| Michael Roberts | HR Director | Human Resources | 96,000 |
| Susan Pretorius | HR Officer | Human Resources | 32,000 |
| Jameel Malik | IT Systems Administrator | IT | 58,000 |
| Thabo Molefe | Network Engineer | IT | 46,000 |
| Nomvula Khumalo | Registered Nurse | Emergency & Trauma | 34,000 |
| Lerato Mokoena | Registered Nurse | Pediatrics | 33,000 |
| Bongani Ndlovu | Registered Nurse | Cardiology | 35,000 |
| Zanele Mahlangu | Nursing Sister | Theatre | 42,000 |
| Kagiso Sithole | Pharmacist | Pharmacy | 61,000 |
| Naledi Zulu | Pharmacy Assistant | Pharmacy | 26,000 |
| Themba Nkosi | Radiographer | Radiology | 44,000 |
| Palesa Radebe | Radiographer | Radiology | 43,000 |
| Deepak Pillay | Lab Technologist | Diagnostics Lab | 41,000 |
| Kavitha Govender | Lab Technician | Diagnostics Lab | 35,000 |
| Dr. Suresh Moodley | Consultant Radiologist | Radiology | 130,000 |
| Dr. Fatima Patel | Pediatrician | Pediatrics | 118,000 |
| Nisha Singh | Physiotherapist | Rehabilitation | 48,000 |
| Dr. Vikram Chetty | Anaesthetist | Theatre | 135,000 |
| David Smith | Facilities Manager | Operations | 52,000 |
| Karen O'Connor | Billing Administrator | Finance | 29,000 |
| James Wilson | Security Supervisor | Operations | 27,000 |
| Linda Fourie | Receptionist | Front Office | 19,000 |
| Peter van Wyk | Procurement Officer | Supply Chain | 38,000 |
| Andile Mbeki | Ward Clerk | Administration | 21,000 |

### Extracted Shareholder Records

| Shareholder Name | Share Percentage | Share Class |
|---|---:|---|
| Dr. Rajesh Naidoo | 18% | Ordinary |
| Cedar Health Holdings (Pty) Ltd | 15% | Ordinary |
| Dr. Johan van der Merwe | 12% | Ordinary |
| Reddy Family Trust | 11% | Ordinary |
| Thabo Molefe | 10% | Ordinary |
| Sarah Botha | 9% | Ordinary |
| Dr. Ahmed Kara | 8% | Preferential |
| Naledi Zulu | 7% | Ordinary |
| Michael Roberts | 6% | Ordinary |
| Dr. Vikram Chetty | 4% | Preferential |

### Attack-Chain Correlation

The Author field discovered in Finding 5 was:

```text
j.malik
```

The database backup identified:

```text
Jameel Malik
IT Systems Administrator
IT Department
```

This connected the metadata discovery with the database records and demonstrated how information from one vulnerability assisted in exploiting another.

### Evidence Screenshot

### Impact

The exposed backup resulted in the disclosure of sensitive employee financial information and corporate ownership information.

An unauthenticated attacker could obtain this information simply by accessing the publicly exposed backup directory.

---

# 4. Full Attack Chain Summary

The following sequence demonstrates the complete path from zero authenticated access to exposure of highly sensitive internal information.

## Step 1 — Reconnaissance

`robots.txt` revealed three hidden/disallowed directories:

```text
/patient
/staff
/old
```

## Step 2 — Username Enumeration

The patient login page displayed different error messages for invalid usernames and incorrect passwords.

This confirmed that `admin` was a valid account.

**Finding 1 — Medium**

## Step 3 — SQL Injection Discovery

A single quote entered into the username field triggered a database syntax error.

This confirmed that the input was vulnerable to SQL injection.

**Finding 2 — Critical**

## Step 4 — Authentication Bypass

Controlled SQL injection testing demonstrated that the login authentication could be bypassed without knowing the password.

**Finding 2 — Critical**

## Step 5 — Patient Report Access

The authenticated portal exposed three confidential patient laboratory reports.

**Finding 3 — High**

## Step 6 — PDF Password Recovery

The weak passwords protecting the PDFs were recovered using authorised password-cracking tools and wordlists.

**Finding 4 — High**

## Step 7 — Metadata Discovery

Metadata extracted from the third report revealed an internal comment pointing to the `/old/` directory.

**Finding 5 — Medium**

## Step 8 — Database Backup Discovery

The `/old/` directory had directory listing enabled and exposed:

```text
mediroza_db_backup_2019.sql
```

**Finding 6 — Critical**

## Step 9 — Sensitive Database Exposure

The database backup contained staff salary records and shareholder information in plain text.

**Finding 7 — Critical**

## Attack Chain Overview

```text
Reconnaissance
      ↓
robots.txt
      ↓
/patient directory
      ↓
Username Enumeration
      ↓
SQL Injection
      ↓
Authentication Bypass
      ↓
Confidential Patient PDFs
      ↓
Weak PDF Passwords
      ↓
PDF Metadata
      ↓
/old/ Directory
      ↓
Database Backup
      ↓
Staff Salaries + Shareholder Data
```

---

# 5. Recommendations and Remediation

## 5.1 Fix Username Enumeration

The login page should return one generic error message for every failed authentication attempt, regardless of whether the username or password was incorrect.

### Recommended Response

```text
Invalid credentials. Please try again.
```

The application should not reveal whether an account exists.

---

## 5.2 Fix SQL Injection

The current login query should be replaced with a parameterised query or prepared statement.

Prepared statements separate SQL code from user-controlled input and prevent crafted input from changing the structure of the SQL query.

### Additional Recommendations

- Use prepared statements throughout the application.
- Never concatenate user input directly into SQL queries.
- Implement server-side input validation.
- Disable detailed database errors in production.
- Use secure password hashing such as Argon2id or bcrypt.
- Apply the principle of least privilege to the database account.

**Priority:** Immediate.

SQL injection is the most critical application vulnerability identified during this assessment.

---

## 5.3 Fix PDF Access and Password Strength

The PDF files should be moved outside the public web root so that they cannot be directly requested through predictable URLs.

Access should instead be controlled through a server-side authorisation mechanism that verifies the user's permissions before serving a document.

### Additional Recommendations

- Enforce server-side access control.
- Verify that the requesting user is authorised to access the specific patient record.
- Avoid predictable filenames for sensitive documents.
- Use strong, unique passwords when document encryption is required.
- Enforce appropriate password complexity requirements.
- Prefer modern document encryption mechanisms where supported.
- Log access to sensitive patient documents.

---

## 5.4 Strip PDF Metadata

Sensitive internal metadata should be removed from documents before they are distributed.

For example:

```bash
exiftool -all= patient_report_3.pdf
```

Organisations should establish a document sanitisation process to remove:

- Internal comments
- Staff usernames
- Author information
- Internal file paths
- Software information
- Unnecessary timestamps
- Other sensitive metadata

Staff should also be trained not to place confidential operational notes inside documents that may leave the organisation.

---

## 5.5 Fix Directory Listing and Remove the Backup

Directory listing should be disabled across the web server.

For Apache, an appropriate configuration can include:

```apache
Options -Indexes
```

The exposed database backup should be removed from the `/old/` directory immediately.

Database backups should never be stored inside the publicly accessible web root.

### Recommended Storage Practices

- Store backups outside the web root.
- Restrict access using operating-system permissions.
- Encrypt sensitive backups at rest.
- Apply access controls to backup storage.
- Maintain a defined backup-retention policy.
- Remove obsolete backups securely.
- Regularly audit web directories for forgotten files.
- Prevent directory listing on all production directories.

---

# 6. Conclusion

This assessment identified a complete attack chain beginning at the patient portal login page and ending with the exposure of highly sensitive internal information.

The attack did not require sophisticated equipment or advanced exploitation techniques. Instead, several common security weaknesses combined to create significant overall risk.

The most serious issues identified were:

- SQL injection allowing authentication bypass
- Unauthorised access to confidential patient reports
- Weak PDF passwords
- Sensitive metadata disclosure
- Public directory listing
- Publicly accessible database backups
- Exposure of employee salary information
- Exposure of shareholder information

The vulnerabilities identified are well-known and have established remediation techniques.

I recommend that the client address all Critical and High severity findings immediately, particularly the SQL injection vulnerability and publicly accessible database backup.

A follow-up security assessment should be conducted after remediation to confirm that the vulnerabilities have been successfully resolved.

---

# 7. Disclaimer

**Submitted by:** Monday Royal Levi 
**Cybersecurity Mentor:** Waqas Karim, CCIE  
**Organisation:** Networkwalks  
**Batch:** B082 | Week 4

This report was produced as part of the Networkwalks B082 Cybersecurity Internship Week 4.

The target was authorised for security testing, and all testing activities were conducted within the agreed scope and with written authorisation from the client.

The techniques documented in this report must never be applied to systems, networks, applications, or data without explicit permission from the owner.

This report contains references to sensitive information obtained during a controlled educational security assessment. It should be handled as **Confidential** and should not be publicly distributed without appropriate redaction and authorisation.

---

# Assessment Completion

| Field | Details |
|---|---|
| **Project** | Web Application Security Assessment |
| **Target** | Mediroza General Hospital |
| **Programme** | Networkwalks Cybersecurity Internship |
| **Batch** | B082 |
| **Project** | Week 4 |
| **Overall Risk** | **CRITICAL** |

---

# End of Report