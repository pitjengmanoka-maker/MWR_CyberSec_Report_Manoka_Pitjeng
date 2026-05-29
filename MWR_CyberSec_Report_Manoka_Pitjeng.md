# Security Assessment Report
## MWR CyberSec — Internship Application Portal

**Assessor:** Manoka Pitjeng  
**TryHackMe Username:** pitjengmanoka  
**Date of Assessment:** 28 May 2026  
**Report Version:** 1.0  

---

## Executive Summary

I tested the MWR CyberSec Internship Application Portal to check for security issues in the web application. My testing focused on the login, registration, user profile, assignment pages, file upload feature, and some API endpoints.

I found **six security findings**, with **two confirmed flags**. The main issue was that the application accepted a modified JWT token, which allowed access to admin functionality. I also found that the application exposed sensitive user information, including a password hash, and accepted unsafe file uploads. I recommend fixing the JWT validation, hiding sensitive data from API responses, and adding stronger file upload checks.

---

## Scope

The assessment was limited to the internship application portal at:

- http://interns.mwrcybersec.loc

Testing was only performed against the provided exam web application and its API endpoints.

---

## Methodology

I first activated the exam using the provided setup link and registered my account on the internship portal. I used my own virtual machine for the assessment because the TryHackMe attack machine had a time limit, and I wanted enough time to test properly without losing my work.

I started by using the application normally in the browser to understand the main features, such as registration, login, profile updates, assignment pages, leaderboard, and file upload.

I then tested the application using browser developer tools, Burp Suite, curl, SQLMap, ffuf, Nikto, and John the Ripper. I used these tools to check authentication, access control, API responses, JWT tokens, duplicate registration, admin endpoints, and file upload behaviour. I only reported issues where I had clear evidence from the testing.

### General Screenshots

**Figure 1: Exam setup completed**

<img width="1120" height="477" alt="exam_setup_redacted" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/exam_setup_redacted.png" />

**Figure 2: Website login page**

<img width="1117" height="477" alt="website_open" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/website_open.png" />

**Figure 3: Burp Suite HTTP history**

<img width="1167" height="476" alt="burp_suit_day1_httpHistory" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/burp_suit_day1_httpHistory.png" />

**Figure 4: Registration successful response**

<img width="1114" height="476" alt="day2_registration_success_response_redacted" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/exam_setup_redacted.png" />

---

## Risk Rating Scheme

All findings in this report are rated using MWR CyberSec's standard risk rating scheme:

| Rating | Definition |
|--------|------------|
| **High** | Potential for an attacker to control, alter, or delete electronic assets. Unauthorized access, data capture, defacement. |
| **Medium** | Combined with other factors, potential for asset compromise. Conditional exploitation, configuration-dependent. |
| **Low** | Likelihood or impact of exploitation is extremely low. Weak ciphers, outdated protocols, programmatic CAPTCHA bypass. |
| **Informational** | Cannot be exploited directly but not aligned with best practice. Information disclosure, version leakage. |

---

## Flags Found

1. MWR{a3fec9f6eae863038537a8c9de0e026b}
2. MWR{1720c8e2a93b5cb2e76e7d4d4a12d97d}

Only two unique flags were confirmed during testing. Other issues were still documented where there was clear evidence, even when no flag was returned.

---

# Findings

### Finding 1: Weak Password Policy

**Risk Rating:** Low

**Flag:** N/A

**Description:**
The application allowed a new user account to be created using a common weak password such as 12345678. This shows that the registration process does not appear to block commonly used passwords or enforce strong password requirements.

**Reproduction Steps:**
1. Sent a registration request to /api/v1/auth/register.
2. Used a common weak password: password.
3. The application accepted the registration request and created the account.
4. Logged in successfully using the same weak password.

**Business Impact:**
Users may create passwords that are easy to guess. This can increase the chance of account compromise, especially if attackers attempt password guessing or credential stuffing.

**Screenshot References:**
- weak_password_registration
  
  <img width="816" height="195" alt="weak_password_registration" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/weak_password_registration.png" />
  
- weak_password_login

  <img width="816" height="195" alt="login_weak_password" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/login_weak_password.png" />  
  

**Remediation:**
The application should enforce stronger password rules. It should block common passwords, require a reasonable minimum length, and use rate limiting or account lockout to reduce password guessing attempts.

## Finding 2: JWT Authentication Bypass Using Unsigned Token

**Risk Rating:** High

**Flag:** MWR{1720c8e2a93b5cb2e76e7d4d4a12d97d}

**Description:**
The application accepted a modified JWT token that used the alg:none algorithm. I was able to change the token claims to include admin-related values and then access an admin endpoint. This shows that the application was not properly verifying the JWT signature before trusting the token.

**Reproduction Steps:**
1. Log in as a normal user and capture the access token.
2. Decode the JWT token to view the header and payload.
3. Modify the JWT header to use alg:none.
4. Add admin-related claims such as role: admin and is_admin: true.
5. Send the modified token to the admin export endpoint /api/v1/admin/export.
6. Observe that the application accepts the token and returns a flag.

**Business Impact:**
An attacker could bypass normal access control and access admin functionality without being an actual admin user. This could expose sensitive applicant information and other internal application data.

**Screenshot References:**

<img width="816" height="195" alt="finding_02_jwt_none_admin_export_flag" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/finding_02_jwt_none_admin_export_flag.png" />


**Remediation:**
The application should reject unsigned JWT tokens and only allow approved signing algorithms. JWT signatures must be verified on every request. The server should not trust role or admin values from a token unless the token is properly signed and validated.

---

## Finding 3: Assignment Start Endpoint Exposes Flag

**Risk Rating:** Medium

**Flag:** MWR{a3fec9f6eae863038537a8c9de0e026b}

**Description:**
The assignment start endpoint returned a flag in the API response when an assignment was started. This exposed sensitive challenge data directly in the response. In a real application, this type of issue could expose information that should only be available after completing a proper action or validation step.

**Reproduction Steps:**
1. Log in as a normal user and capture the access token.
2. Send a request to start Assignment 1 using /api/v1/assignments/1/start.
3. Observe the API response.
4. The response contains the flag value directly.

**Business Impact:**
A user could access sensitive assignment information earlier than intended. If similar behaviour existed in a real-world system, it could expose protected workflow data or internal values before the user is meant to see them.

**Screenshot References:**

<img width="768" height="295" alt="finding_01_assignment_start_flag_response_redacted" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/finding_01_assignment_start_flag_response_redacted.png" />

**Remediation:**
The application should not return sensitive values such as flags or internal challenge data in normal start responses. The backend should only return information needed by the frontend, and sensitive values should be protected behind the correct validation logic.

---

## Finding 4: Sensitive User Data Disclosure During Duplicate Registration

**Risk Rating:** High

**Flag:** N/A

**Description:**
When I tried to register an account using an email address that was already registered, the application returned more information than necessary. The response included sensitive user details such as the user ID, role, application status, timestamps, and the password_hash field. The application should only return a simple message that the email already exists, without exposing the existing user's full details.

**Reproduction Steps:**
1. Register a user account normally.
2. Send another registration request using the same email address.
3. Observe the API response.
4. The response shows the existing user's details, including a password hash.
5. The leaked bcrypt hash was also tested using John the Ripper with rockyou.txt, but it was not cracked within the assessment time.

**Business Impact:**
Leaking password hashes is a serious security risk because an attacker can try to crack the hash offline. The response also exposes personal and internal account information that should not be shown to users. This could lead to privacy issues and make other attacks easier.

**Screenshot References:**

<img width="1912" height="256" alt="finding_sensitive_user_data_leak_duplicate_register" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/finding_sensitive_user_data_leak_duplicate_register_redacted.png" />

**Remediation:**
The registration endpoint should return a generic error message such as `Email already registered`. It should not return the full user object or sensitive fields. Password hashes, roles, IDs, and application status values should never be included in public API error responses.

---

## Finding 5: CV Upload Accepts Non-PDF Files

**Risk Rating:** Medium

**Flag:** N/A

**Description:**
The CV upload functionality accepted a non-PDF file during testing. The file was uploaded successfully and the server returned a success message. This means the backend was not properly enforcing file type validation.

**Reproduction Steps:**
1. Log in as a normal user.
2. Create a test file that is not a PDF, such as test.txt.
3. Upload the file to the CV upload endpoint.
4. Observe that the server accepts the file and returns a successful upload response.

**Business Impact:**
If file uploads are not properly restricted, an attacker may be able to upload unsafe files. Depending on how the files are stored and served, this could lead to malware upload, stored content abuse, or further server-side risks.

**Screenshot References:**

<img width="417" height="227" alt="finding_cv_upload_non_pdf_accepted" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/finding_cv_upload_non_pdf_accepted.png" />


**Remediation:**
The application should only allow approved file types, such as PDF files, for CV uploads. Validation should happen on the server side using file extension checks, MIME type checks, and file signature checks. Uploaded files should be renamed safely, stored outside the web root where possible, and scanned before being made available.

---

## Finding 6: Server Version Disclosure and Missing Security Headers

**Risk Rating:** Informational

**Flag:** N/A

**Description:**
The web server response disclosed the server software and version as `nginx/1.24.0 (Ubuntu)`. The response also did not show some common security headers such as Content Security Policy and other browser protection headers. This does not directly exploit the application, but it gives attackers extra information about the environment.

**Reproduction Steps:**
1. Send a normal request to the home page.
2. Review the response headers in Burp Suite or using curl.
3. Observe the `Server` header and the missing security headers.

**Business Impact:**
Server version information can help an attacker understand the technology stack and search for known weaknesses. Missing security headers can also reduce browser-side protection against some client-side attacks.

**Screenshot References:**

<img width="867" height="376" alt="home_response_headers" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/home_response_headers.png" />

**Remediation:**
The server should hide detailed version information where possible. Security headers such as Content Security Policy, X-Frame-Options or frame-ancestors, X-Content-Type-Options, Referrer-Policy, and Permissions-Policy should be reviewed and added where suitable.

---

## Finding 7: Refresh Token Stored in Browser Local Storage

**Risk Rating:** Low

**Flag:** N/A

**Description:**
The application stored the refresh token in browser local storage. I observed this using the browser developer tools. Local storage can be accessed by JavaScript running in the page, so if a cross-site scripting issue exists anywhere in the application, an attacker could potentially steal the refresh token.

**Reproduction Steps:**
1. Log in to the application normally.
2. Open browser developer tools.
3. Go to the Application tab and view Local Storage for the site.
4. Observe that the refresh_token value is stored there.
5. Review the frontend JavaScript and observe the refresh endpoint being used by the client.

**Business Impact:**
If an attacker finds a way to run JavaScript in the user's browser, the stored refresh token could be stolen and used to maintain access. This increases the risk of account takeover in combination with another client-side vulnerability.

**Screenshot References:**
<img width="1905" height="882" alt="day2_localstorage_refresh_token_observed" src="https://github.com/pitjengmanoka-maker/MWR_CyberSec_Report_Manoka_Pitjeng/blob/main/screenshots/day2_localstorage_refresh_token_observed_redacted.png" />

**Remediation:**
Refresh tokens should be protected using secure storage methods. A safer option is to store refresh tokens in secure, HttpOnly, SameSite cookies so JavaScript cannot directly access them. Tokens should also have proper expiry, rotation, and revocation controls.

---

## Conclusion

The most important issue found was the JWT authentication bypass, because it allowed admin functionality to be accessed using a modified unsigned token. The duplicate registration data leak was also serious because it exposed sensitive user data, including a password hash. I was also able to register using a weak password such as 12345678 which meant that the web server's password policy needs to be changed. 

The application should prioritise fixing JWT validation, reducing sensitive information in API responses, and improving upload validation. The lower-risk issues, such as server header disclosure and refresh token storage, should also be reviewed as part of improving the overall security posture of the application.
