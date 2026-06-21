<KB: mysql max connections>

## Introduction

**Problem Statement:**
You can change `max_connections` while MySQL is running via `SET`.

**Environment:
- Technology / Service: MySQL
- Component: mysqld
- Severity: P1/P2/P3 based on impact

---

## Process

### 1. Identifying the Problem
 <!-- TODO: add detail here -->

### 2. Diagnosis

```bash
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