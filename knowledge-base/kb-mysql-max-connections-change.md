# KB: MySQL max connections change
## Introduction
**Problem Statement:**
You can change `max_connections` while MySQL is running via `SET`

**Environment:
- Technology / Service: MySQL
- Component: mysqld, port 3306
- Severity: P1/P2/P3 based on impact
---
## Process
### 1. Identifying the Problem
<!-- TODO: add detail here -->

### 2. Diagnosis
<How was it found — alert, user report, log spike. Use exact details from input.>
```
mysql> SET GLOBAL max_connections = 5000;
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW VARIABLES LIKE "max_connections";
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 5000  |
+-----------------+-------+
1 row in set (0.00 sec)
```
<what the output showed and why it mattered>

### 3. Root Cause
One clear paragraph. State exactly what was broken and why — not "a network issue" but the specific cause.

### 4. Fix Applied
Exact fix. Include the real command, config change, or code.
```
<!-- TODO: add exact fix command -->
```
### 5. Verification
<How was it confirmed fixed. Include real command and expected output.>
```
<!-- TODO: add verification command -->
```
---
## Conclusion
**Summary:**
<2-3 sentences: what broke, what fixed it, outcome>
**Prevention:
- <specific actionable practice 1>
- <specific actionable practice 2>
- <specific actionable practice 3>
---
## Tags
`<tag1>` `<tag2>` `<tag3>`