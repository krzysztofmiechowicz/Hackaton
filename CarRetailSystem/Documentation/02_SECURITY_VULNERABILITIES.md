# Security Vulnerabilities

All vulnerabilities identified through static code analysis of the CarRetailSystem legacy codebase.

---

## CRITICAL — Production Blocking

### 1. SQL Injection (Pervasive)

**Affected files and exact locations:**

| File | Line(s) | Vulnerable code |
|------|---------|-----------------|
| `ASP/pages/login.asp` | 28 | `"... WHERE UserName='" & username & "' AND Password='" & password & "'"` |
| `ASP/pages/inventory.asp` | 59–66 | INSERT with all form fields concatenated directly |
| `VB6Components/CarInventory.vb` | 50, 54 | `" AND Make = '" & make & "'"` and `" AND Price <= " & maxPrice` |
| `VB6Components/CarInventory.vb` | 70 | `"SELECT * FROM Cars WHERE CarID = " & carID` |
| `VB6Components/CarInventory.vb` | 83, 120 | UPDATE and INSERT with concatenated carID, quantity |
| `VB6Components/SalesOrder.vb` | 59–60 | Full INSERT into Sales with all parameters concatenated |
| `VB6Components/SalesOrder.vb` | 78, 82 | UPDATE Cars and INSERT Inventory_Log concatenated |
| `VB6Components/SalesOrder.vb` | 105 | `"... WHERE S.CustomerID = " & customerID` |

**Impact:** Full database takeover, data exfiltration, data destruction, authentication bypass.

**Example attack vector on login.asp:28:**
```
username: ' OR '1'='1
password: anything
→ logs in as the first user in the database (admin)
```

**Fix:** Use parameterized queries or call the existing stored procedures that are already parameterized (`sp_GetAvailableCars`, `sp_CreateSalesOrder`, etc.).

---

### 2. Hardcoded Database Credentials

**Affected locations:**

| File | Line | Credential |
|------|------|-----------|
| `ASP/includes/config.asp` | 7 | `UID=sa;PWD=Admin@123;` |
| `VB6Components/CarInventory.vb` | 25 | `UID=sa;PWD=Admin@123;` |
| `VB6Components/SalesOrder.vb` | 23 | `UID=sa;PWD=Admin@123;` |

**Account:** `sa` (SQL Server System Administrator — highest privilege level)

**Impact:** Anyone with access to source code or decompiled DLLs has full administrative control over the database server.

**Fix:** Use Windows Authentication (no credentials in code), or load credentials from environment variables / secrets manager (Azure Key Vault, HashiCorp Vault).

---

### 3. Hardcoded SMTP Credentials

| File | Line | Value |
|------|------|-------|
| `ASP/includes/config.asp` | 31 | `SMTP_PASSWORD = "EmailPassword123"` |

**Impact:** Email account compromise, potential spam relay abuse.

**Fix:** Load from environment variable or secrets manager.

---

### 4. Plain Text Password Storage

**Location:** `SQL/database_setup.sql` (Users table definition, lines 228–233 sample data)

**Evidence:** Password column stores values like `admin123`, `pass123` directly.

**Impact:** Any database dump, backup, or log exposes all user passwords. Compromising one system compromises users on other systems (password reuse).

**Fix:** Hash passwords with bcrypt, PBKDF2, or Argon2 with per-user salt. Never store reversible or unhashed passwords.

---

### 5. Authentication Bypass via SQL Injection

Combines vulnerabilities #1 and #4. The login query at `login.asp:28` can be bypassed entirely without knowing any password, granting attacker access as any user including admin.

---

## HIGH — Fix Soon

### 6. Cross-Site Scripting (XSS)

**Affected locations:**

| File | Line | Issue |
|------|------|-------|
| `ASP/pages/index.asp` | 40 | `Response.Write Session("UserName")` — no encoding |
| `ASP/pages/inventory.asp` | 107 | Database values written to output without HTMLEncode |

**Impact:** Malicious scripts injected into the browser — session token theft, keylogging, phishing overlays.

**Fix:** Wrap all output with `HTMLEncode()` — the function already exists in `utils.asp`.

---

### 7. No CSRF Protection

**Affected location:** `ASP/pages/inventory.asp` form (line 80)

**Impact:** A malicious website can forge POST requests (add/delete cars) on behalf of a logged-in user.

**Fix:** Generate and validate a per-session CSRF token in every state-changing form.

---

### 8. Session Hijacking Risk

**Issues in `ASP/includes/session.asp`:**
- Session cookie (`CarRetailSID`) transmitted over plain HTTP (no Secure flag)
- No `HttpOnly` flag — cookie accessible via JavaScript (XSS can steal it)
- No session fixation protection on login
- IP address stored but never validated

**Impact:** Session token theft via network sniffing or XSS leads to account takeover.

**Fix:** Enforce HTTPS, set `Secure` and `HttpOnly` flags, regenerate session ID on login.

---

### 9. No HTTPS Enforcement

**Impact:** All credentials, session tokens, and sensitive data transmitted in plaintext over the network.

**Fix:** Install TLS certificate, configure IIS to redirect HTTP → HTTPS, enforce `HSTS`.

---

### 10. No Rate Limiting on Login

**Location:** `ASP/pages/login.asp`

**Impact:** Brute force and credential stuffing attacks can run unchecked.

**Fix:** Lock account or introduce delay after N failed attempts (e.g., 5). Log attempts.

---

### 11. Sensitive Data Leaked in Logs

**Location:** `ASP/includes/security.asp:52–76` (`LogEvent` function)

**Issue:** Logs written to a plain file; content may include usernames, actions, and error details revealing system internals.

**Fix:** Use structured logging to a secure sink (never log passwords or tokens). Restrict log file access.

---

### 12. Insufficient Input Sanitization

**Location:** `ASP/includes/security.asp:38–48`

```vbscript
Function SanitizeInput(inputStr)
    SanitizeInput = Trim(inputStr)   ' only trims whitespace
End Function
```

**Impact:** Provides false sense of security. Does not prevent SQL injection or XSS.

**Fix:** Sanitization alone cannot prevent SQL injection — use parameterized queries. Use `HTMLEncode()` for XSS prevention on output.

---

### 13. Weak Password Policy

**Location:** `ASP/includes/security.asp:104–111`

```vbscript
Function ValidatePassword(password)
    ValidatePassword = (Len(password) >= 6)
End Function
```

**Impact:** Passwords as short as 6 characters, no complexity requirements.

**Fix:** Enforce minimum 12 characters, require uppercase, lowercase, digit, and symbol.

---

## MEDIUM — Design Issues

### 14. Error Suppression Hides Failures

**Pattern used throughout:** `On Error Resume Next`

**Locations:** `inventory.asp:68`, `index.asp:69`, `CarInventory.vb:44`

**Impact:** Silent failures — database errors, failed inserts, and connection issues are swallowed. Users see no feedback; operators have no visibility.

**Fix:** Remove `On Error Resume Next`. Implement structured error handling with appropriate user messages and logging.

---

### 15. Demo Credentials Hardcoded in HTML

**Location:** `ASP/pages/login.asp:86–89`

HTML form pre-fills `admin`/`admin123` as hints. Exposes admin credentials to anyone viewing page source.

**Fix:** Remove demo hints from production builds.

---

### 16. No Distributed Session

Sessions stored in IIS in-process memory. Cannot load-balance across multiple servers; server restart logs out all users.

---

### 17. No Encryption at Rest

Database backups and log files contain plain text passwords and sensitive customer data.

---

## Vulnerability Summary

| Severity | Count | Examples |
|----------|-------|---------|
| Critical | 5 | SQL injection, hardcoded sa credentials, plain text passwords |
| High | 8 | XSS, CSRF, session hijacking, no HTTPS, no rate limiting |
| Medium | 4 | Error suppression, no distributed session, weak password policy |
| **Total** | **17** | |

---

## OWASP Top 10 Coverage

| OWASP Category | Present |
|----------------|---------|
| A01 Broken Access Control | Partial (no CSRF, role bypass via SQLi) |
| A02 Cryptographic Failures | Yes (plain text passwords, no HTTPS) |
| A03 Injection | Yes (SQL injection — pervasive) |
| A04 Insecure Design | Yes (monolith, no security layers) |
| A05 Security Misconfiguration | Yes (hardcoded credentials, sa account) |
| A06 Vulnerable Components | Yes (deprecated VB6, IIS 5, SQL Server 7) |
| A07 Identification & Auth Failures | Yes (plain text, no MFA, no lockout) |
| A08 Software & Data Integrity | Partial (no CSRF) |
| A09 Logging & Monitoring Failures | Yes (silent `On Error Resume Next`) |
| A10 SSRF | Not applicable |
