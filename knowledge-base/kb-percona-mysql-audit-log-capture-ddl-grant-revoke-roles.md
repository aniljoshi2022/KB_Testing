# KB: Capturing DDL, Connections, Failed Logins and Grant/Revoke Privileges in Percona MySQL 8.4.7 Audit Log

## Introduction

**Problem Statement:**
When configuring Percona MySQL 8.4.7's `audit_log_filter` plugin to capture security-relevant events (DDL, connections, failed logins, GRANT/REVOKE), role-based grant and revoke statements were silently dropped from the audit log despite the filter appearing correct. The standard `grant` and `revoke` sql_command_id values do not cover role assignments, which use separate identifiers.

**Environment:**
- Technology / Service: Percona MySQL 8.4.7, audit_log_filter plugin
- Component: `audit_log_filter_set_filter()`, audit log JSON filter definition
- Severity: P2 (security audit gap)

---

## Process

### 1. Identifying the Problem
The user configured an audit log filter to capture connections, failed logins, DDL statements, and GRANT/REVOKE privilege changes. After applying the filter and running `audit_log_filter_flush()`, GRANT/REVOKE statements on direct privileges were captured correctly — but granting or revoking roles to/from users produced no audit log entries.

### 2. Diagnosis

The original filter included `grant` and `revoke` under the `query` class:

```json
{ "field": { "name": "sql_command_id", "value": "grant" } },
{ "field": { "name": "sql_command_id", "value": "revoke" } }
```

This captured direct privilege grants like:
```sql
GRANT SELECT ON spice.* TO example@'%';
```

But role assignments like the following produced no audit log entry:
```sql
GRANT role_name TO user@'%';
REVOKE role_name FROM user@'%';
```

The key insight: in Percona MySQL 8.4.7, role-based GRANT/REVOKE operations are internally classified under different `sql_command_id` values — `grant_role` and `revoke_role` — not `grant` and `revoke`.

Additionally, an `authorization` class block was tested but found unnecessary for SQL-level role capture. However, the `authorization` class does capture server-level privilege changes that occur outside of SQL text, providing more complete coverage.

### 3. Root Cause

Percona MySQL 8.4.7 distinguishes between privilege grants (`grant`) and role grants (`grant_role`) at the internal command classification level. The `audit_log_filter` plugin uses these internal `sql_command_id` values for matching, so a filter with only `grant` and `revoke` will never match role assignment statements. These are treated as separate command types.

### 4. Fix Applied

Add `grant_role` and `revoke_role` to the `or` block inside the `query` class filter:

```json
{ "field": { "name": "sql_command_id", "value": "grant_role" } },
{ "field": { "name": "sql_command_id", "value": "revoke_role" } }
```

Full working filter:

```sql
SELECT audit_log_filter_set_filter('capture_core_events', '{
  "filter": {
    "class": [
      {
        "name": "connection",
        "event": [
          { "name": "connect" },
          { "name": "disconnect" },
          { "name": "failed_login" }
        ]
      },
      {
        "name": "query",
        "event": [
          {
            "name": "start",
            "log": {
              "or": [
                { "field": { "name": "sql_command_id", "value": "create_db" } },
                { "field": { "name": "sql_command_id", "value": "drop_db" } },
                { "field": { "name": "sql_command_id", "value": "alter_db" } },
                { "field": { "name": "sql_command_id", "value": "truncate" } },
                { "field": { "name": "sql_command_id", "value": "create_table" } },
                { "field": { "name": "sql_command_id", "value": "alter_table" } },
                { "field": { "name": "sql_command_id", "value": "drop_table" } },
                { "field": { "name": "sql_command_id", "value": "create_user" } },
                { "field": { "name": "sql_command_id", "value": "alter_user" } },
                { "field": { "name": "sql_command_id", "value": "drop_user" } },
                { "field": { "name": "sql_command_id", "value": "create_role" } },
                { "field": { "name": "sql_command_id", "value": "drop_role" } },
                { "field": { "name": "sql_command_id", "value": "grant_role" } },
                { "field": { "name": "sql_command_id", "value": "revoke_role" } },
                { "field": { "name": "sql_command_id", "value": "grant" } },
                { "field": { "name": "sql_command_id", "value": "revoke" } }
              ]
            }
          }
        ]
      }
    ]
  }
}');
```

After applying, always flush the filter:

```sql
SELECT audit_log_filter_flush();
```

### 5. Verification

After applying the updated filter and flushing, run a role grant and check the audit log:

```sql
GRANT role_name TO user@'%';
```

Expected audit log entry:

```json
{
  "timestamp": "2026-01-23 10:00:00",
  "class": "query",
  "event": "query_start",
  "query_data": {
    "query": "grant role_name to user@'%'",
    "status": 0,
    "sql_command": "grant_role"
  }
}
```

The `sql_command` field in the audit log entry should show `grant_role` or `revoke_role`, confirming the filter is now capturing role assignments.

---

## Conclusion

**Summary:**
Percona MySQL 8.4.7 classifies role-based GRANT/REVOKE operations under `grant_role` and `revoke_role` sql_command_id values, separate from direct privilege `grant` and `revoke`. Adding these two identifiers to the audit_log_filter query class resolves the gap. Optionally, adding an `authorization` class block provides additional coverage for server-level privilege changes that occur outside SQL text.

**Prevention:**
- Always include both `grant`/`revoke` AND `grant_role`/`revoke_role` in audit filters that need to capture all privilege changes
- Run `SELECT audit_log_filter_flush()` after every filter change — changes are not applied until flushed
- Test filters explicitly with role assignments, direct grants, DDL, and connection events after any filter modification
- Consider adding the `authorization` class block for the most complete privilege audit coverage
- Document your filter definition in version control so changes are tracked over time

---

## Tags

`percona` `mysql` `audit-log` `DDL` `grant` `revoke` `roles` `security`
