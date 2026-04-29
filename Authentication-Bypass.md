# Authentication Bypass — Writeup

**Platform:** TryHackMe  
**Category:** Web Application Security  
**Difficulty:** Easy/Medium  
**Vulnerability Type:** Authentication Bypass via URL Manipulation  
**OWASP Category:** Broken Access Control (A01:2021)  

---

## Vulnerability Overview

Authentication Bypass is a vulnerability that allows an attacker to access restricted areas of a web application without valid credentials. This can occur through various techniques including:

- URL manipulation (direct object access)
- Parameter tampering
- Cookie manipulation
- Forced browsing to restricted pages

In this case, the application failed to enforce authentication checks server-side, allowing direct access to the admin panel by simply navigating to its URL.

---

## Target Overview

The target was a web application with:
- A public-facing login page
- A hidden admin panel at a non-obvious URL path
- No proper server-side session validation on protected routes

---

## Reconnaissance

Started by browsing the application and mapping its structure.

Observed the login page at:
```
http://<target-ip>/login
```

Performed directory enumeration using Gobuster to discover hidden paths:

```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt
```

Discovered the admin panel path:
```
/admin
/admin/dashboard
/admin/panel
```

---

## Exploitation

### Step 1 — Direct URL Access (Forced Browsing)

Without logging in, directly navigated to the admin panel URL:

```
http://<target-ip>/admin/dashboard
```

**Result:** The application loaded the admin panel without redirecting to the login page.

The server only hid the link from the UI — it never verified whether the user had an authenticated session before serving the admin page.

### Step 2 — URL Parameter Manipulation

In some areas of the application, the URL contained a role or access parameter:

```
http://<target-ip>/dashboard?role=user
```

Modified the parameter value:

```
http://<target-ip>/dashboard?role=admin
```

**Result:** The application granted admin-level access based solely on the URL parameter — no server-side verification was performed.

### Step 3 — Admin Panel Access Confirmed

Successfully accessed the admin panel which included:
- User management controls
- Application configuration settings
- All registered user data

---

## Why This Works

The application relied on **client-side controls** to restrict access. The navigation menu simply did not display the admin link to regular users — but the server never checked whether the requesting user had admin privileges.

```
Regular User visits /admin/dashboard
        │
        ▼
Server receives GET /admin/dashboard
        │
        ▼
No session/role check performed server-side
        │
        ▼
Server returns admin page to any requester
        │
        ▼
Full admin access granted
```

This is a fundamental design flaw — **security through obscurity is not security.**

---

## Real-World Impact

In a production environment, this vulnerability could allow an attacker to:

- Gain full administrative control over the application
- Access, modify, or delete all user accounts and data
- Change application configuration and security settings
- Escalate to further attacks (RCE, data exfiltration, backdoor installation)
- Completely compromise the application and its users

---

## Remediation

| Action | Details |
|--------|---------|
| Server-Side Auth Checks | Every protected route must verify the user's session and role on the server |
| Session Validation | Check session token on every request — not just at login |
| Role-Based Access Control | Implement proper RBAC — do not rely on URL parameters for role determination |
| Redirect on Failure | Unauthenticated requests to protected routes must redirect to login |
| Security Testing | Include forced browsing and parameter tampering in every web app assessment |

**Secure Code Example:**

```python
# Vulnerable — no auth check
@app.route('/admin/dashboard')
def admin_dashboard():
    return render_template('admin.html')

# Secure — server-side session and role check
@app.route('/admin/dashboard')
def admin_dashboard():
    if not session.get('logged_in') or session.get('role') != 'admin':
        return redirect('/login'), 403
    return render_template('admin.html')
```

---

## Key Takeaways

- Never rely on hiding URLs as a security measure — always enforce authentication server-side.
- Directory enumeration (Gobuster, FFUF) is essential during recon — hidden pages are not protected pages.
- URL parameters should never determine access level — roles must be stored and verified server-side via session.
- Authentication bypass vulnerabilities are simple to find but carry critical severity in real engagements.
- Always test every discovered endpoint — even those not linked in the UI — for unauthorized access.

---

*Written by Muhammad Muzammil Khan — Penetration Tester*  
*TryHackMe Profile: https://tryhackme.com/r/p/mmuzammilkhan*
