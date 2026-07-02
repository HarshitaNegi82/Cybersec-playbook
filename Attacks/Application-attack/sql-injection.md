# SQL Injection

SQL injection is an attack where malicious SQL code is inserted into an input field that gets incorporated into a database query without proper validation. The database executes the attacker's code as part of the query, potentially exposing or modifying data it should not.

---

## Which OSI Layer and Why

SQL injection is a **Layer 7 (Application)** attack. It exploits how the web application constructs SQL queries using user-supplied input. The attack travels inside a normal HTTP request on port 80 or 443 — from a network perspective, the traffic is completely legitimate.

A network firewall operates at Layers 3–4. It sees source IP, destination IP, protocol, and port. It sees valid TCP on port 443 going to a web server. It has no visibility into the HTTP request body, and even if it did, it would not understand what constitutes a dangerous SQL fragment. This is why a network firewall alone cannot stop SQL injection.

---

## How It Works

A login form sends username and password to the server. The server constructs a query:

```python
# Vulnerable code
query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'"
```

Normal input: `username = alice`, `password = secret123`

Resulting query:
```sql
SELECT * FROM users WHERE username = 'alice' AND password = 'secret123'
```

Malicious input: `username = ' OR '1'='1' --`, `password = anything`

Resulting query:
```sql
SELECT * FROM users WHERE username = '' OR '1'='1' --' AND password = 'anything'
```

`--` is a SQL comment — everything after it is ignored. `'1'='1'` is always true. The query returns all users. Authentication is bypassed.

---

## What Attackers Can Do

- **Authentication bypass** — log in without valid credentials
- **Data extraction** — dump entire database tables using UNION SELECT
- **Data modification** — UPDATE or DELETE records
- **Command execution** — on misconfigured databases, execute OS commands via SQL (e.g., `xp_cmdshell` in Microsoft SQL Server)
- **Schema enumeration** — discover table names, column names, database version

---

## Real Fix — Parameterized Queries

The actual defense is at the code level. Never concatenate user input into SQL strings.

```python
# Safe — parameterized query
cursor.execute("SELECT * FROM users WHERE username = ? AND password = ?", (username, password))
```

With parameterized queries, the database treats user input strictly as data — never as SQL code. The structure of the query is fixed before any user input is substituted in. There is no way to change the query structure through input.

---

## Additional Defenses

**Web Application Firewall (WAF):**  
A WAF inspects HTTP traffic at Layer 7 and can detect common SQL injection patterns (UNION SELECT, OR 1=1, comment sequences). It is a detection and blocking layer — not a substitute for parameterized queries, but a useful additional control.

**Least privilege on the database account:**  
The database user used by the web application should have only the permissions it needs (SELECT on specific tables). Not DROP TABLE, not TRUNCATE, not shell execution.

**Input validation:**  
Reject inputs that contain characters that have no legitimate use for that field — a username field does not need single quotes or SQL keywords.

**Error handling:**  
Never return raw database error messages to users. They reveal table names, column names, and database version — exactly what an attacker needs for further exploitation.
