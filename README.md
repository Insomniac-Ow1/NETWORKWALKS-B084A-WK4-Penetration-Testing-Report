# Penetration Testing Report: Mediroza General Hospital

| Field              | Detail                                                                 |
| -------------------- | ------------------------------------------------------------------------- |
| **Client**         | Mediroza General Hospital                                              |
| **Target**         | `https://medirozahospital.com`                                        |
| **Engagement Type** | Black-box Penetration Test                                            |
| **Duration**       | 5 Days                                                                 |
| **Batch**          | B083 | Week 4                                                        |
| **Pentester**      | Omar Amin                                                              |
| **Authorization**  | Written permission granted by the client for all phases of this engagement, as part of the Networkwalks Cybersecurity & Ethical Hacking internship (educational lab target; all patient, staff and shareholder data referenced below is synthetic test data) |

---

## 01. Executive Summary

A 5-day black-box penetration test was conducted against Mediroza General Hospital's web infrastructure (`https://medirozahospital.com`). The objective was to evaluate the security posture of the site by identifying vulnerabilities and exploiting them to demonstrate real impact, working through four phases: gaining initial access to confidential patient data, cracking the encryption protecting that data, uncovering a further critical data exposure on the server, and documenting everything in this report.

The assessment confirmed **critical vulnerabilities**. A single, well-known SQL injection string bypassed authentication entirely and returned three other patients' confidential pathology reports with no valid credentials. Those reports were protected only by weak, dictionary-crackable passwords, each recovered in well under a minute. Separately, a forgotten database backup file was found publicly exposed on the server, disclosing the full names, job titles, emails, phone numbers, national ID numbers and **monthly salaries of all 30 hospital staff**, together with the hospital's complete **shareholder register**. Taken together, these findings mean an anonymous visitor to the site could, within minutes and using only freely available tools, obtain protected health information and sensitive financial/HR data belonging to the organization.

## 02. Scope and Methodology

**Scope:** Testing was limited strictly to `https://medirozahospital.com`. No denial-of-service testing and no social engineering were performed, per the agreed rules of engagement.

**Reconnaissance & Enumeration Tools Used:**

| Tool       | Purpose                                                                 |
| ------------ | -------------------------------------------------------------------------- |
| `whois`    | Domain registration, registrar (NameCheap, Inc.) and privacy-proxy details |
| `nslookup` | Resolved the target domain to its IP address                           |
| `whatweb`  | Server headers, HTTP status codes and web technology fingerprinting    |
| `curl -I`  | Raw HTTP response headers                                              |
| `wafw00f`  | Web Application Firewall detection                                     |
| `dnsrecon` | General DNS enumeration (A, MX, TXT/SPF, DMARC, SRV records)           |

**Other tools used:** a web browser for manual authentication and input-validation testing; the OnlineHashCrack PDF hash extractor; and the Networkwalks Password Cracker (dictionary-attack lab tool) for credential recovery.

**Methodology (by phase):**

1. **Phase 1 — Initial Access:** Passive/active reconnaissance, identification of exposed entry points via `robots.txt`, and authentication testing (including SQL injection) against the login forms.
2. **Phase 2 — Data Extraction:** Encryption analysis of the retrieved PDF files and dictionary-based password cracking to recover their contents.
3. **Phase 3 — Critical Data Exposure:** Deeper analysis of the server and exposed directories, uncovering a further, more severe data exposure beyond the original three files.
4. **Phase 4 — Reporting:** Consolidating findings, evidence and risk ratings into this report.

**Limitations:** This was a time-boxed, black-box assessment; it demonstrates exploitable vulnerabilities but is not an exhaustive audit of every endpoint on the site.

## 03. Findings and Proof of Exploitation

### Reconnaissance & Information Gathering Summary

- **Target infrastructure:** Resolved to IP `199.188.201.16`, hosted via NameCheap (`dns1.namecheaphosting.com`, `dns2.namecheaphosting.com`), with mail routed through Jellyfish Systems (`mx1/2/3-hosting.jellyfish.systems`). Server banner: **LiteSpeed**.
- **WAF status:** `wafw00f` identified the built-in LiteSpeed WAF; no additional third-party WAF was detected. As shown below, this WAF did not block the SQL injection payload used in testing.
- **DNS posture:** SPF ends in a soft fail (`~all`); DMARC is published but set to `p=none` (monitor-only, no enforcement); DNSSEC is not enabled.
- **Exposed endpoints (`robots.txt`):**

  ```text
  User-agent: *
  Disallow: /patient/
  Disallow: /staff/
  Disallow: /old/
  ```

  `robots.txt` is a crawler convention, not an access control — publishing these paths effectively handed an attacker a shortlist of sensitive areas to investigate first. All three were investigated directly (Phases 1 and 3 below).

> Full command outputs and screenshots for this stage are saved in `1. reconnaissance/txt/` and `1. reconnaissance/screenshots/`.

### Phase 1 — Initial Access & Patient Report Retrieval

**Authentication testing:** Both the Staff Login (`/staff/login.php`) and Patient Portal login (`/patient/login.php`) were tested for SQL injection using the classic authentication-bypass payload `admin'--` in the username/ID field (any value in the password field).

![Staff login SQL injection attempt](3.%20sql%20injection/1.%20trying%20sql%20injection.png)

*Staff Login: `admin'--` submitted as the Staff ID.*

![Patient portal login SQL injection attempt](3.%20sql%20injection/2.%20trying%20sql%20injection%20on%20patient%20login.png)

*Patient Portal: the same payload submitted as the Username, with "Show password" enabled to confirm the exact characters sent.*

**Result — authentication bypassed:** Submitting the payload against the Patient Portal login successfully logged in as `admin` with no valid password, returning a list of three patients' password-protected pathology reports.

![Successful SQL injection giving access to three patient PDF files](3.%20sql%20injection/3.%20sql%20injection%20on%20patient%20login%20successful%2C%20got%20access%20to%20the%203%20pdf%20files.png)

*"My lab reports" page reached via the injection payload, listing Pathology Report – S. Dlamini, P. Reddy, and E. Thompson.*

**Why it matters:** This is the most serious finding in this engagement. A single, well-known SQL injection string typed into a login field — with no valid credentials, no account lockout, and no MFA in the way — returned confidential patient health records belonging to other people. If the application builds its query by concatenating user input directly into SQL (e.g. `SELECT * FROM patients WHERE id='<input>' AND password='<input>'`), this payload comments out the password check entirely.

**Deliverable:** The three retrieved, originally password-protected PDF files (`patient_report_1.pdf`, `patient_report_2.pdf`, `patient_report_3.pdf`) are provided alongside this report.

### Phase 2 — Cracking File Encryption

**Hash extraction:** Each PDF's password hash was extracted with the OnlineHashCrack PDF hash extractor (`pdf2john`/`pdf2hashcat`-compatible `$pdf$...` format).

![PDF hash extraction](4.%20cracking%20retrieved%20files/1.%20Used%20OnlineHashcrack%20pdf%20hash%20extractor%20tool%20to%20get%20the%20hashes%20for%20the%203%20files.png)

**Dictionary attack:** Using the Networkwalks Password Cracker, `patient_report_2.pdf`'s password (`password`) was recovered in 2 attempts against the tool's built-in 100-word list.

![Cracking patient_report_2.pdf](4.%20cracking%20retrieved%20files/2.%20used%20netwrokwalks%20tool%20to%20crack%20the%20passwords%20using%20the%20hashes.png)

`patient_report_1.pdf` was cracked the same way (password: `123456`). `patient_report_3.pdf` did not fall to the built-in list, so a larger wordlist (`JTR_default_password.txt`, 3,556 entries) was used, recovering its password (`!@#$%^&`) on attempt 3,546 of 3,556 — confirming that a single approach does not always work, and a wider dictionary was needed for the more complex password.

![Cracking patient_report_3.pdf with a larger wordlist](4.%20cracking%20retrieved%20files/3.%20Used%20networkealks%20tool%20to%20crack%20password%20of%20pdf3.png)

**Passwords recovered:**

| File                   | Password Recovered |
| ------------------------ | --------------------- |
| `patient_report_1.pdf` | `123456`            |
| `patient_report_2.pdf` | `password`          |
| `patient_report_3.pdf` | `!@#$%^&`           |

**Recovered contents:** All three reports opened successfully, each a one-page pathology laboratory report with patient name, date of birth, patient ID, referring doctor, specimen type, lab reference number and flagged test results.

![Example recovered pathology report content](4.%20cracking%20retrieved%20files/content%20of%20pdfs/pdf1.png)

| File                   | Patient (synthetic) | Lab Ref No.  | Notable Flagged Results                                  |
| ------------------------ | --------------------- | -------------- | ----------------------------------------------------------- |
| `patient_report_1.pdf` | Sipho Dlamini        | LR-2024-1187 | White Cell Count — HIGH                                   |
| `patient_report_2.pdf` | Priya Reddy          | LR-2024-1192 | Total Cholesterol, LDL Cholesterol, Triglycerides — HIGH |
| `patient_report_3.pdf` | Emily Thompson       | LR-2024-1205 | Haemoglobin, Ferritin, Vitamin D — LOW                    |

**Why it matters:** All three passwords were trivially weak — a sequential number, a dictionary word, and a short symbol string — and every one was cracked in seconds to minutes with free tools. A password meant to protect confidential medical results should never be guessable this quickly.

**Deliverable:** Recovered contents of all 3 files, with proof of successful access, are shown above; extracted hashes are saved in `4. cracking retrieved files/hashes/`.

### Phase 3 — Critical Data Exposure: Staff Salaries & Shareholder Details

Following the hint to examine everything retrieved so far beyond the obvious content, the `/old/` path flagged by `robots.txt` was investigated directly (directory listing was enabled):

![Directory listing of /old/ exposing a SQL backup](2.%20checking%20domains%20from%20the%20robots%20txt/-old%20has%20a%20sql%20backup.png)

*`https://medirozahospital.com/old/` listing `mediroza_db_backup_2019.sql` (7 KB), with no authentication required to download it.*

The backup contained two tables: **`staff`** (30 records) and **`shareholders`** (10 records). The following is the full readable summary of the confidential data uncovered, as required for this phase's deliverable.

**Hospital Employee Salaries (full list — 30 staff):**

| # | Name                     | Job Title                 | Department          | Monthly Salary (ZAR) | Date Joined |
| - | -------------------------- | ---------------------------- | ---------------------- | ----------------------: | ------------- |
| 1 | Dr. Rajesh Naidoo         | Chief Pathologist          | Diagnostics Lab      | 138,000                | 2009-03-16  |
| 2 | Sarah Botha               | Chief Financial Officer    | Finance              | 152,000                | 2011-07-01  |
| 3 | Dr. Johan van der Merwe   | Medical Director           | Management           | 160,000                | 2007-01-22  |
| 4 | Dr. Anita Naicker         | Consultant Cardiologist    | Cardiology           | 132,000                | 2012-09-10  |
| 5 | Dr. Ahmed Kara            | Consultant Physician       | Internal Medicine    | 128,000                | 2013-02-18  |
| 6 | Dr. Yusuf Cassim          | Senior Registrar           | Emergency & Trauma   | 74,000                 | 2018-05-04  |
| 7 | Michael Roberts           | HR Director                | Human Resources      | 96,000                 | 2010-11-15  |
| 8 | Susan Pretorius           | HR Officer                 | Human Resources      | 32,000                 | 2016-08-23  |
| 9 | Jameel Malik              | IT Systems Administrator   | IT                   | 58,000                 | 2015-04-12  |
| 10 | Thabo Molefe              | Network Engineer           | IT                   | 46,000                 | 2017-10-02  |
| 11 | Nomvula Khumalo           | Registered Nurse           | Emergency & Trauma   | 34,000                 | 2016-01-19  |
| 12 | Lerato Mokoena            | Registered Nurse           | Pediatrics           | 33,000                 | 2017-06-07  |
| 13 | Bongani Ndlovu            | Registered Nurse           | Cardiology           | 35,000                 | 2015-12-01  |
| 14 | Zanele Mahlangu           | Nursing Sister             | Theatre              | 42,000                 | 2013-03-25  |
| 15 | Kagiso Sithole            | Pharmacist                 | Pharmacy             | 61,000                 | 2014-09-08  |
| 16 | Naledi Zulu               | Pharmacy Assistant         | Pharmacy             | 26,000                 | 2019-02-14  |
| 17 | Themba Nkosi              | Radiographer                | Radiology            | 44,000                 | 2016-07-30  |
| 18 | Palesa Radebe             | Radiographer                | Radiology            | 43,000                 | 2017-04-11  |
| 19 | Deepak Pillay             | Lab Technologist           | Diagnostics Lab      | 41,000                 | 2015-05-20  |
| 20 | Kavitha Govender          | Lab Technician             | Diagnostics Lab      | 35,000                 | 2018-08-06  |
| 21 | Dr. Suresh Moodley        | Consultant Radiologist     | Radiology            | 130,000                | 2012-02-28  |
| 22 | Dr. Fatima Patel          | Pediatrician                | Pediatrics           | 118,000                | 2013-10-17  |
| 23 | Nisha Singh               | Physiotherapist            | Rehabilitation       | 48,000                 | 2016-11-09  |
| 24 | Dr. Vikram Chetty         | Anaesthetist                | Theatre              | 135,000                | 2011-06-13  |
| 25 | David Smith               | Facilities Manager         | Operations           | 52,000                 | 2014-01-27  |
| 26 | Karen O'Connor            | Billing Administrator      | Finance              | 29,000                 | 2018-03-19  |
| 27 | James Wilson              | Security Supervisor        | Operations           | 27,000                 | 2019-09-02  |
| 28 | Linda Fourie              | Receptionist                | Front Office         | 19,000                 | 2020-02-10  |
| 29 | Peter van Wyk             | Procurement Officer        | Supply Chain         | 38,000                 | 2015-08-24  |
| 30 | Andile Mbeki              | Ward Clerk                  | Administration       | 21,000                 | 2019-11-05  |

*Each staff record in the raw backup also includes a work email, phone number and national ID number (e.g. record #1: `r.naidoo@medirozahospital.com`, `+27 82 101 2007`, ID `85021013011081`) — omitted here field-by-field for brevity; the complete 30-record set is preserved as evidence in `2. checking domains from the robots txt/backup copy.sql`.*

**Shareholder Register (full list — 10 shareholders):**

| # | Shareholder                      | Share % | Shares Held | Share Class   |
| - | ----------------------------------- | --------: | ------------: | --------------- |
| 1 | Dr. Rajesh Naidoo                 | 18.0%   | 180,000     | Ordinary      |
| 2 | Cedar Health Holdings (Pty) Ltd   | 15.0%   | 150,000     | Ordinary      |
| 3 | Dr. Johan van der Merwe           | 12.0%   | 120,000     | Ordinary      |
| 4 | Reddy Family Trust                | 11.0%   | 110,000     | Ordinary      |
| 5 | Thabo Molefe                      | 10.0%   | 100,000     | Ordinary      |
| 6 | Sarah Botha                       | 9.0%    | 90,000      | Ordinary      |
| 7 | Dr. Ahmed Kara                    | 8.0%    | 80,000      | Preferential  |
| 8 | Naledi Zulu                       | 7.0%    | 70,000      | Ordinary      |
| 9 | Michael Roberts                   | 6.0%    | 60,000      | Ordinary      |
| 10 | Dr. Vikram Chetty                 | 4.0%    | 40,000      | Preferential  |

**Why it matters:** A single forgotten backup, hinted at by `robots.txt` and served with directory listing enabled, exposed the complete payroll of the organization and its full ownership structure — including the fact that the Chief Pathologist and Medical Director are also the two largest individual shareholders. This is sensitive HR and corporate-governance data that should never be reachable from the public internet, let alone discoverable in two steps from the homepage.

**Deliverable:** The documented evidence (Figure above, raw backup file, and the readable summary tables above) fulfils this phase's requirement in full.

## 04. Risk Rating

| Finding                                                    | Vulnerability Type                            | Risk Level   | Justification                                                                                                           |
| ------------------------------------------------------------- | -------------------------------------------------- | -------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Reconnaissance / `robots.txt` disclosure                    | Information Disclosure                        | **Low**      | Standard DNS/WHOIS/robots.txt exposure; aided scoping but presented no direct risk on its own.                        |
| Phase 1 — SQL Injection & Broken Access Control             | Authentication Bypass                         | **Critical** | Allowed an unauthenticated attacker to bypass login entirely and retrieve other patients' confidential health records. |
| Phase 2 — Weak PDF Encryption Passwords                     | Weak Credential / Weak Encryption Password    | **High**     | All three "lab-issued" passwords were cracked in seconds to minutes using free, publicly available dictionary tools.  |
| Phase 3 — Exposed Legacy Database Backup                    | Sensitive Data Exposure / Backup Disclosure   | **Critical** | A public, unauthenticated backup exposed complete staff PII, salaries and the full shareholder register.               |
| Missing modern HTTP security headers                        | Security Misconfiguration                     | **Low**      | No HSTS, CSP, `X-Frame-Options` or `X-Content-Type-Options` observed; a defense-in-depth gap rather than a direct exploit. |

**Overall engagement risk: Critical.** The combination of an authentication-bypass vulnerability and a publicly exposed backup file means an unauthenticated attacker can obtain both protected health information and sensitive HR/financial data with minimal effort.

## 05. Recommendations and Remediation

1. **Fix the SQL injection (highest priority).** Replace all string-concatenated SQL in login and query logic with parameterized queries / prepared statements (or a vetted ORM), across both the Staff Login and Patient Portal. Use generic "invalid credentials" error messages that do not reveal whether a username exists.
2. **Enforce strict, server-side access control.** `/patient/`, `/staff/` and any portal scripts must validate a real authenticated session and role before returning data — never rely on `robots.txt` or an obscure URL as a substitute for access control.
3. **Remove the exposed backup and change how backups are handled.** Delete `/old/` from the web root immediately. Store database backups outside the publicly served directory, encrypted at rest, with access limited to authorized administrators, and disable directory listing site-wide.
4. **Replace weak, predictable PDF passwords with a proper secure-delivery process.** Stop issuing short dictionary words or symbol strings as "lab-issued" passwords; use long, random, per-recipient passwords delivered through a separate channel, or move report delivery into an authenticated portal with per-session access control and audit logging.
5. **Treat all exposed data as compromised.** Since staff PII, salaries and three patients' health records were accessed, rotate any credentials that may have been in the backup and follow the organization's data-breach notification process for affected staff and patients.
6. **Tune the WAF for SQL injection and add rate limiting.** The detected LiteSpeed WAF did not block the injection payload used in this test; enable and test SQLi-specific rule sets, and add account lockout/rate limiting on login forms.
7. **Add standard HTTP security headers** (HSTS, CSP, `X-Frame-Options`, `X-Content-Type-Options`) and tighten the SPF/DMARC email-authentication posture (move DMARC from `p=none` toward enforcement once monitoring confirms it is safe).
8. **Re-test after remediation** and build recurring, authorized penetration testing into the development lifecycle rather than treating this as a one-time exercise.

---

## Deliverables Submitted

| Phase                              | Required Deliverable                                                   | Provided                                                                                         |
| ------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Phase 1 — Initial Access            | Proof of access and the 3 retrieved PDF files                          | Screenshots above + `patient_report_1.pdf`, `patient_report_2.pdf`, `patient_report_3.pdf` (attached) |
| Phase 2 — Data Extraction           | Recovered contents of all 3 files with proof of successful access      | Screenshots and recovered-content table above                                                    |
| Phase 3 — Critical Data Exposure    | Full documented evidence and a readable summary of the data uncovered  | Directory-listing screenshot, raw backup file, and full salary/shareholder tables above           |
| Phase 4 — Pentest Report            | A complete professional penetration testing report                     | This document                                                                                     |

-End-
