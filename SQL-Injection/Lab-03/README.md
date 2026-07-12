### SQL injection attack, querying the database type and version on Oracle

**Category:** SQL Injection  
**Difficulty:** Practitioner  
**Platform:** PortSwigger Web Security Academy

### Overview

This lab demonstrates a `UNION`-based SQL injection vulnerability in a product category filter that allows an attacker to 
extract arbitrary data from the back-end database — in this case, the Oracle database's own version banner — without any
authentication or special privileges. The objective was to make the database return the exact Oracle version string 
(`Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production...`).
![Lab overview - not solved yet](images/lab3overview.png)

### Vulnerability

The category filter takes user input and inserts it directly into an SQL query, likely similar to:
```sql
SELECT name, description FROM products WHERE category = 'Lifestyle' AND released = 1
```
Since the `category` parameter is concatenated directly into the query without sanitization or parameterized statements, SQL syntax can be injected to append an entirely separate `SELECT` statement via `UNION`, letting an attacker pull back arbitrary data instead of product listings.

### Methodology

1. **Identify the Injection Point**
   With Burp Suite's proxy running, browsing to the "Lifestyle" category fired off this request:
   ```
   GET /filter?category=Lifestyle
   ```
   ![Intercepted GET request in Burp Proxy](images/lab3-getreq.png)
   The `category` parameter is the obvious candidate — it drives a `WHERE` clause on the server, making it a classic injection point.

2. **Determine the Number of Columns**
   Before extracting data with `UNION SELECT`, the number of columns must match the original query, or the database throws an error. Oracle also requires a `FROM` clause on every `SELECT`, so `dual` (Oracle's dummy one-row table) is used to satisfy that requirement:
   ```
   GET /filter?category=Lifestyle'+union+select+null,null+from+dual--
   ```
   ![Testing column count with UNION SELECT in Repeater](images/lab3-trial1.png)
   This returned `200 OK` and rendered normally, confirming the query expects **two columns** and that the injection point is exploitable.

3. **Inject the Version Query**
   With the column count confirmed, one `NULL` placeholder was replaced with a query against Oracle's `v$version` view, which stores the version banner in a column called `banner`:
   ```
   GET /filter?category=Lifestyle'+union+select+null,banner+from+v$version--
   ```
   ![Querying v$version banner column](images/lab3-trial2.png)
   The response came back as `200 OK` with a noticeably larger body (Content-Length rose from ~9.3KB to ~10.1KB), indicating the banner string had been embedded in the page.

4. **Confirm Success**
   The crafted URL was copied from Repeater and loaded directly in the browser:
   ![Copying the crafted URL from Repeater](images/lab3-url.png)
   The page loaded correctly, and the Oracle version banner appeared in place of a product entry, confirming the injection worked as intended.

### Why This Works

1. The application builds SQL queries using string concatenation instead of parameterized queries (prepared statements).
2. The single quote (`'`) closes the `category` string early.
3. The `UNION SELECT` clause appends a second query that returns attacker-chosen data — here, the Oracle version banner from `v$version`.
4. The SQL comment sequence (`--`) causes the database to ignore the remainder of the original query, so it doesn't interfere with the injected `SELECT`.
5. Because the application renders whatever the query returns directly onto the page, the injected data (the version banner) is displayed to the attacker.

The lab flipped over to solved once the payload was confirmed in the browser:
![Lab solved](images/lab3-solved.png)

### Impact

An attacker can:
1. Extract arbitrary data from the database, including sensitive tables not intended to be exposed.
2. Fingerprint the database engine and exact version, which helps in crafting further, version-specific attacks.
3. Use the same technique to enumerate table names, column names, and eventually sensitive data such as credentials.
4. Potentially escalate to full database compromise if additional injection points or higher-privileged database accounts are present.

### Remediation

1. Use parameterized queries (prepared statements) for all database interactions instead of concatenating user input into SQL queries.
2. Apply the principle of least privilege to the database account used by the application.
3. Use input validation and allow-lists where appropriate.
4. Use an ORM or query builder that automatically parameterizes queries.
5. Implement a Web Application Firewall (WAF) as an additional layer of defense.

### Payload Summary
```sql
' UNION SELECT NULL,NULL FROM dual--          -- confirms column count
' UNION SELECT NULL,banner FROM v$version--    -- extracts Oracle version string
```
