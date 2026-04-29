# IDOR (Insecure Direct Object Reference) — Writeup

**Platform:** TryHackMe  
**Category:** Web Application Security  
**Difficulty:** Easy  
**Vulnerability Type:** IDOR (Insecure Direct Object Reference)  
**OWASP Category:** Broken Access Control (A01:2021)  

---

## Vulnerability Overview

Insecure Direct Object Reference (IDOR) is a type of access control vulnerability that occurs when an application uses user-controllable input to access objects directly — without verifying whether the user is authorized to access them.

In simple terms: if you can change a number in a URL and access someone else's data, the application is vulnerable to IDOR.

IDOR falls under **Broken Access Control**, which is ranked **#1 in the OWASP Top 10 (2021)**.

---

## Target Overview

The target was a **shopping website** that allowed users to:
- Create accounts
- View their orders
- Access their profile and personal information

Each user had a unique numeric ID that the application used to fetch their data.

---

## Reconnaissance

Logged into the application with a test account and started browsing functionality.

Navigated to the user profile/order page and observed the URL structure:

```
http://<target-ip>/profile?user_id=1234
```

The `user_id` parameter was directly visible in the URL — an immediate red flag.

Opened **Burp Suite** to intercept and analyze the requests being made to the server.

---

## Exploitation

### Step 1 — Identify the Parameter

While browsing the profile page, the following GET request was captured in Burp Suite:

```
GET /profile?user_id=1234 HTTP/1.1
Host: <target-ip>
Cookie: session=<session-token>
```

The `user_id` value was a simple numeric ID with no additional authorization token or signature.

### Step 2 — Manipulate the Parameter

Manually changed the `user_id` value in the URL to another user's ID:

```
http://<target-ip>/profile?user_id=1233
```

### Step 3 — Access Another User's Data

The server returned the **profile and personal data of a completely different user** — without any authorization check.

Accessed data included:
- Username
- Email address
- Order history
- Personal details

**No authentication bypass was needed — simply changing the ID was enough.**

---

## Why This Works

The application trusted the `user_id` value supplied by the user without verifying:
- Does this session belong to user 1233?
- Is this user authorized to view this profile?

The server fetched data directly from the database using the supplied ID — no server-side authorization check was in place.

```
User Request (user_id=1233)
        │
        ▼
Server receives ID
        │
        ▼
Database Query: SELECT * FROM users WHERE id = 1233
        │
        ▼
Data returned — NO authorization check performed
        │
        ▼
Attacker sees victim's data
```

---

## Real-World Impact

In a real production environment, this vulnerability could allow an attacker to:

- Enumerate all user accounts by iterating through IDs
- Access sensitive personal data (PII) of all users
- View private order history and payment information
- Potentially modify or delete other users' data (if POST/PUT requests are also vulnerable)
- Violate data protection regulations (GDPR, PCI-DSS)

---

## Remediation

| Action | Details |
|--------|---------|
| Server-side Authorization | Always verify that the logged-in user owns the requested resource |
| Use Indirect References | Replace numeric IDs with unpredictable tokens (UUID/GUID) |
| Session Validation | Cross-check user_id against the authenticated session on every request |
| Access Control Tests | Include IDOR test cases in every web application security assessment |

**Secure Code Example (Pseudocode):**

```python
# Vulnerable
user_data = db.query("SELECT * FROM users WHERE id = ?", request.user_id)

# Secure
if request.user_id != session.authenticated_user_id:
    return 403 Forbidden
user_data = db.query("SELECT * FROM users WHERE id = ?", session.authenticated_user_id)
```

---

## Key Takeaways

- IDOR is one of the most common and impactful vulnerabilities in web applications.
- Always test numeric IDs, GUIDs, filenames, and any user-supplied reference to an object.
- Authorization must be enforced **server-side** — client-side controls are always bypassable.
- This vulnerability is easy to find but often missed in code reviews — making it a high-value target during pentests.
- During real engagements, always test IDOR on: profile pages, order/invoice endpoints, file downloads, and API endpoints.

---

*Written by Muhammad Muzammil Khan — Penetration Tester*  
*TryHackMe Profile: https://tryhackme.com/r/p/mmuzammilkhan*
