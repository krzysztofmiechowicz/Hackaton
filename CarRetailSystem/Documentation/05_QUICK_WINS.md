# Quick Wins

Immediate fixes that can be applied to the **existing legacy codebase today** — before the full migration begins. These reduce risk without requiring new infrastructure.

Each fix is self-contained and can be applied in under an hour.

---

## 1. Replace String-Concat SQL with Stored Procedures (Critical)

**Problem:** SQL injection in `login.asp:28` and `inventory.asp:59-66` via direct string concatenation.

**Why this fix is safe:** The stored procedures already exist in the database and are already parameterized. This is a usage fix, not a rewrite.

### login.asp — Before

```vbscript
sql = "SELECT UserID, UserName, Role FROM Users WHERE UserName='" & username & "' AND Password='" & password & "'"
Set rs = conn.Execute(sql)
```

### login.asp — After

```vbscript
Set cmd = Server.CreateObject("ADODB.Command")
cmd.ActiveConnection = conn
cmd.CommandText = "SELECT UserID, UserName, Role FROM Users WHERE UserName=? AND Password=?"
cmd.CommandType = 1
cmd.Parameters.Append cmd.CreateParameter("@UserName", 200, 1, 50, username)
cmd.Parameters.Append cmd.CreateParameter("@Password", 200, 1, 100, password)
Set rs = cmd.Execute()
```

### inventory.asp — Before

```vbscript
sql = "INSERT INTO Cars (Make, Model, Year, ...) VALUES ('" & _
      Request.Form("Make") & "', '" & Request.Form("Model") & "', " & _
      Request.Form("Year") & ", ...)"
conn.Execute(sql)
```

### inventory.asp — After

```vbscript
Set cmd = Server.CreateObject("ADODB.Command")
cmd.ActiveConnection = conn
cmd.CommandText = "INSERT INTO Cars (Make, Model, Year, Color, VIN, Price, Stock, CreatedDate) VALUES (?,?,?,?,?,?,?,GETDATE())"
cmd.CommandType = 1
cmd.Parameters.Append cmd.CreateParameter("@Make",  200, 1, 50,  Request.Form("Make"))
cmd.Parameters.Append cmd.CreateParameter("@Model", 200, 1, 50,  Request.Form("Model"))
cmd.Parameters.Append cmd.CreateParameter("@Year",   5,  1, 4,   CInt(Request.Form("Year")))
cmd.Parameters.Append cmd.CreateParameter("@Color", 200, 1, 30,  Request.Form("Color"))
cmd.Parameters.Append cmd.CreateParameter("@VIN",   200, 1, 17,  Request.Form("VIN"))
cmd.Parameters.Append cmd.CreateParameter("@Price",  6,  1, 10,  CDbl(Request.Form("Price")))
cmd.Parameters.Append cmd.CreateParameter("@Stock",  3,  1, 4,   CInt(Request.Form("Stock")))
cmd.Execute()
```

Also apply the same pattern in both VB6 files (`CarInventory.vb` and `SalesOrder.vb`) — replace `conn.Execute(sql)` calls with `ADODB.Command` and explicit parameters.

---

## 2. Move Credentials to Environment Variables (Critical)

**Problem:** `config.asp:7` has `PWD=Admin@123` (SQL Server sa account) and `config.asp:31` has `SMTP_PASSWORD = "EmailPassword123"` in plain text.

### config.asp — Before

```vbscript
DB_CONNECTION_STRING = "Provider=SQLOLEDB;Server=.\SQLEXPRESS;Database=CarRetailDB;UID=sa;PWD=Admin@123;"
SMTP_PASSWORD = "EmailPassword123"
```

### config.asp — After

```vbscript
DB_CONNECTION_STRING = GetEnvironmentVariable("CAR_RETAIL_DB_CONN")
SMTP_PASSWORD        = GetEnvironmentVariable("CAR_RETAIL_SMTP_PWD")

Function GetEnvironmentVariable(name)
    Dim wshShell
    Set wshShell = Server.CreateObject("WScript.Shell")
    GetEnvironmentVariable = wshShell.ExpandEnvironmentStrings("%" & name & "%")
    Set wshShell = Nothing
End Function
```

Then set the environment variables in IIS application pool advanced settings or in Windows System Environment Variables. Never commit credentials to source control.

**Also apply in VB6 components:** Replace the hardcoded `DB_CONN_STRING` constant in `CarInventory.vb:25` and `SalesOrder.vb:23` with a call to read from an environment variable or Windows Registry (safer than source code).

---

## 3. Enforce HTTPS in IIS (Critical)

1. Install a TLS certificate (Let's Encrypt via win-acme, or internal CA)
2. In IIS Manager → site → SSL Settings → check "Require SSL"
3. Add HTTP → HTTPS redirect rule to `web.config`:

```xml
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="HTTP to HTTPS" stopProcessing="true">
          <match url="(.*)" />
          <conditions>
            <add input="{HTTPS}" pattern="^OFF$" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

---

## 4. Encode All Output to Prevent XSS (High)

**Problem:** `index.asp:40` and `inventory.asp:107` write user-controlled values directly to HTML output.

**Fix:** The `HTMLEncode()` function already exists in `utils.asp`. Use it on every `Response.Write` that outputs user data or database content.

### Before

```vbscript
Response.Write "Welcome, " & Session("UserName")
Response.Write "<td>" & rs("Make") & "</td>"
```

### After

```vbscript
Response.Write "Welcome, " & HTMLEncode(Session("UserName"))
Response.Write "<td>" & HTMLEncode(rs("Make")) & "</td>"
```

Apply this to every `Response.Write` in `index.asp`, `inventory.asp`, and `login.asp`.

---

## 5. Remove Demo Credentials from HTML (High)

**Problem:** `login.asp:86-89` shows `admin / admin123` in the page source as helpful hints.

**Fix:** Delete those lines. Any demonstration should use a staging environment, not production.

---

## 6. Replace `On Error Resume Next` with Error Logging (Medium)

**Problem:** `On Error Resume Next` in `inventory.asp:68`, `index.asp:69`, `CarInventory.vb:44` silently swallows all errors.

**Fix:** Remove the suppression and replace with an error handler that logs and shows a safe message:

```vbscript
' Remove: On Error Resume Next

conn.Execute(sql)
If Err.Number <> 0 Then
    LogEvent "ERROR", "Database error: " & Err.Description
    Response.Write "<p>An error occurred. Please try again.</p>"
    Err.Clear
    Response.End
End If
```

---

## 7. Add Minimum Input Validation on Forms (High)

The existing `SanitizeInput()` in `security.asp` only trims. Add numeric validation for numeric fields:

```vbscript
' Before inserting Year, Price, Stock from inventory.asp form:
Dim carYear, carPrice, carStock
carYear  = Request.Form("Year")
carPrice = Request.Form("Price")
carStock = Request.Form("Stock")

If Not IsNumeric(carYear) Or Not IsNumeric(carPrice) Or Not IsNumeric(carStock) Then
    Response.Write "Invalid input. Year, Price and Stock must be numeric."
    Response.End
End If

carYear  = CInt(carYear)
carPrice = CDbl(carPrice)
carStock = CInt(carStock)
```

---

## Impact Summary

| Fix | Time Estimate | Vulnerability Closed |
|-----|--------------|----------------------|
| Parameterized queries (all pages + VB6) | 3–4 hours | SQL injection (#1) |
| Move credentials to env vars | 1 hour | Hardcoded credentials (#2, #3) |
| Enforce HTTPS in IIS | 1 hour | No encryption in transit (#9) |
| Add HTMLEncode to all outputs | 1 hour | XSS (#6) |
| Remove demo credentials from HTML | 5 minutes | Credential exposure (#15) |
| Replace On Error Resume Next | 2 hours | Silent error suppression (#14) |
| Add numeric input validation | 1 hour | Partial SQL injection protection |

**Total estimated time: ~10 hours**

These fixes eliminate the most critical attack vectors and can be applied before the full migration begins. They do not require any new infrastructure, new dependencies, or downtime.
