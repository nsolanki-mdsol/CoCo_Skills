---
name: account-inventory
description: |
  Snowflake account inventory and statistics for administration, audits, and capacity planning.

  Topics:
    Users - users, user count, accounts, service accounts, person users, user type, legacy service
    Databases - databases, db count
    Warehouses - warehouses, compute, wh count
    Roles - roles, role count, RBAC
    Schemas - schemas, schema count
    Tables - tables, table count, objects
    Reader Accounts - reader, reader accounts, managed accounts
    Shares - shares, inbound shares, outbound shares, data sharing, share count
    Resource Monitors - resource monitor, monitors, credit quota
    Network Policies - network policy, network policies, IP whitelist, allowed IPs
    Tags - tags, tag count, tag references, tagged objects, sensitive data, PII, classification, tag values
    Object Types - object types, objects by database, objects by schema, procedures, functions, stages, streams, tasks, pipes
    Iceberg Tables - iceberg, iceberg tables, iceberg count, iceberg by database, iceberg by schema, catalog integration, snowflake managed, externally managed, glue, polaris
    Summary - summary, inventory, all counts
---

# Account Inventory Skill

## Purpose
Quick account-level inventory and statistics for Snowflake administration.

## When to Use
- Account audits
- Capacity planning
- RBAC reviews
- Documentation needs

---

## Quick Summary (All Counts)

```sql
SELECT 
    (SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.USERS WHERE deleted_on IS NULL) AS active_users,
    (SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES WHERE deleted IS NULL) AS databases,
    (SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES WHERE deleted IS NULL) AS warehouses,
    (SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES WHERE deleted_on IS NULL) AS roles,
    (SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.SCHEMATA WHERE deleted IS NULL) AS schemas,
    (SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL) AS tables;
```

---

## Individual Counts

### Users Count
```sql
-- Active users count
SELECT COUNT(*) AS user_count 
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS 
WHERE deleted_on IS NULL;

-- Users by status
SELECT 
    CASE WHEN disabled = 'true' THEN 'Disabled' ELSE 'Enabled' END AS status,
    COUNT(*) AS count
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
GROUP BY 1;

-- Users by type (PERSON, SERVICE, LEGACY_SERVICE)
SELECT 
    COALESCE(type, 'LEGACY (NULL)') AS user_type,
    COUNT(*) AS count
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
GROUP BY type
ORDER BY count DESC;

-- Users by type and status combined
SELECT 
    COALESCE(type, 'LEGACY (NULL)') AS user_type,
    CASE WHEN disabled = 'true' THEN 'Disabled' ELSE 'Enabled' END AS status,
    COUNT(*) AS count
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
GROUP BY type, disabled
ORDER BY type, status;

-- Service accounts only
SELECT name, login_name, email, owner, created_on, last_success_login, disabled
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND type = 'SERVICE'
ORDER BY created_on DESC;

-- Person (human) users only
SELECT name, login_name, email, owner, created_on, last_success_login, disabled
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND type = 'PERSON'
ORDER BY created_on DESC;

-- Legacy service accounts (created before user types feature)
SELECT name, login_name, email, owner, created_on, last_success_login, disabled
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND type = 'LEGACY_SERVICE'
ORDER BY created_on DESC;

-- Users with details
SELECT name, login_name, email, type, created_on, last_success_login, disabled
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
ORDER BY created_on DESC;
```

### Databases Count
```sql
-- Database count
SELECT COUNT(*) AS database_count 
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES 
WHERE deleted IS NULL;

-- Databases with details
SELECT database_name, database_owner, created, last_altered, retention_time
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES
WHERE deleted IS NULL
ORDER BY created DESC;

-- Real-time count
SELECT COUNT(*) AS database_count FROM INFORMATION_SCHEMA.DATABASES;
```

### Warehouses Count
```sql
-- Warehouse count
SELECT COUNT(*) AS warehouse_count 
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES 
WHERE deleted IS NULL;

-- Warehouses with details
SELECT warehouse_name, warehouse_size, warehouse_type, auto_suspend, auto_resume, owner
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
WHERE deleted IS NULL
ORDER BY warehouse_name;

-- Real-time using SHOW
SHOW WAREHOUSES;
```

### Roles Count
```sql
-- Role count
SELECT COUNT(*) AS role_count 
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES 
WHERE deleted_on IS NULL;

-- Roles by type (account vs database)
SELECT 
    CASE WHEN is_database_role = 'YES' THEN 'Database Role' ELSE 'Account Role' END AS role_type,
    COUNT(*) AS count
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES
WHERE deleted_on IS NULL
GROUP BY 1;

-- Roles with details
SELECT name, owner, created_on, granted_to_roles, granted_roles
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES
WHERE deleted_on IS NULL
ORDER BY created_on DESC;
```

### Schemas Count
```sql
-- Schema count
SELECT COUNT(*) AS schema_count 
FROM SNOWFLAKE.ACCOUNT_USAGE.SCHEMATA 
WHERE deleted IS NULL;

-- Schemas by database
SELECT catalog_name AS database_name, COUNT(*) AS schema_count
FROM SNOWFLAKE.ACCOUNT_USAGE.SCHEMATA
WHERE deleted IS NULL
GROUP BY 1
ORDER BY 2 DESC;
```

### Tables Count
```sql
-- Table count
SELECT COUNT(*) AS table_count 
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES 
WHERE deleted IS NULL;

-- Tables by database
SELECT table_catalog AS database_name, COUNT(*) AS table_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
GROUP BY 1
ORDER BY 2 DESC;

-- Tables by type
SELECT table_type, COUNT(*) AS count
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
GROUP BY 1
ORDER BY 2 DESC;
```

### Reader Accounts Count
```sql
-- Reader accounts count
SHOW MANAGED ACCOUNTS;
SELECT COUNT(*) AS reader_account_count FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Reader accounts with details
SHOW MANAGED ACCOUNTS;
SELECT "name", "cloud", "region", "locator", "created_on", "url" 
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Shares to reader accounts (outbound shares)
SHOW SHARES;
SELECT "name", "kind", "database_name", "owner", "to" 
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "kind" = 'OUTBOUND';
```

### Shares Count
```sql
-- All shares count (inbound and outbound)
SHOW SHARES;
SELECT COUNT(*) AS total_shares FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Shares by type (inbound vs outbound)
SHOW SHARES;
SELECT 
    "kind" AS share_type,
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "kind"
ORDER BY count DESC;

-- Outbound shares (shared by this account)
SHOW SHARES;
SELECT 
    "name" AS share_name,
    "database_name",
    "owner",
    "to" AS shared_to,
    "created_on"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "kind" = 'OUTBOUND'
ORDER BY "created_on" DESC;

-- Inbound shares (shared to this account)
SHOW SHARES;
SELECT 
    "name" AS share_name,
    "database_name",
    "owner",
    "created_on"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "kind" = 'INBOUND'
ORDER BY "created_on" DESC;

-- Outbound shares with consumer details
SHOW SHARES;
SELECT 
    "name" AS share_name,
    "database_name",
    "to" AS consumers,
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "kind" = 'OUTBOUND';

-- Objects in a specific share (replace <SHARE_NAME>)
-- DESCRIBE SHARE <SHARE_NAME>;

-- Databases created from inbound shares
SELECT 
    database_name,
    database_owner,
    origin AS share_origin,
    created
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES
WHERE deleted IS NULL
  AND origin IS NOT NULL
ORDER BY created DESC;
```

### Resource Monitors Count
```sql
-- Resource monitors count
SHOW RESOURCE MONITORS;
SELECT COUNT(*) AS resource_monitor_count FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Resource monitors with details
SHOW RESOURCE MONITORS;
SELECT "name", "credit_quota", "used_credits", "remaining_credits", 
       "level", "frequency", "start_time", "end_time",
       "notify_at", "suspend_at", "suspend_immediately_at"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Resource monitors usage percentage
SHOW RESOURCE MONITORS;
SELECT "name", "credit_quota", "used_credits",
       ROUND(("used_credits" / NULLIF("credit_quota", 0)) * 100, 2) AS usage_pct,
       "suspend_at", "suspend_immediately_at"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
ORDER BY usage_pct DESC;
```

### Network Policies Count
```sql
-- Network policies count
SHOW NETWORK POLICIES;
SELECT COUNT(*) AS network_policy_count FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Network policies with details
SHOW NETWORK POLICIES;
SELECT "name", "created_on", "comment", "entries_in_allowed_ip_list", 
       "entries_in_blocked_ip_list", "entries_in_allowed_network_rules",
       "entries_in_blocked_network_rules"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Network policy assignments (account level)
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.POLICY_REFERENCES(
    POLICY_NAME => '<policy_name>',
    POLICY_KIND => 'NETWORK_POLICY'
));

-- Check account-level network policy
SHOW PARAMETERS LIKE 'network_policy' IN ACCOUNT;

-- Network policies assigned to users
SELECT u.name AS user_name, 
       u.default_warehouse,
       p.value AS network_policy
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS u
LEFT JOIN TABLE(FLATTEN(input => PARSE_JSON('[]'))) p -- placeholder
WHERE u.deleted_on IS NULL
ORDER BY u.name;
```

### Iceberg Tables Count
```sql
-- Total Iceberg tables count
SELECT COUNT(*) AS iceberg_table_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
  AND is_iceberg = 'YES';

-- Iceberg tables by database
SELECT 
    table_catalog AS database_name,
    COUNT(*) AS iceberg_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
  AND is_iceberg = 'YES'
GROUP BY table_catalog
ORDER BY iceberg_count DESC;

-- Iceberg tables by database and schema
SELECT 
    table_catalog AS database_name,
    table_schema AS schema_name,
    COUNT(*) AS iceberg_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
  AND is_iceberg = 'YES'
GROUP BY table_catalog, table_schema
ORDER BY iceberg_count DESC;
```

### Iceberg Tables by Catalog Type (Snowflake-Managed vs External)
```sql
-- Count Iceberg tables by catalog type (SNOWFLAKE vs GLUE/POLARIS/etc)
-- Uses SHOW command for real-time catalog info
SHOW ICEBERG TABLES IN ACCOUNT;
SELECT 
    "catalog" AS catalog_type,
    CASE 
        WHEN "catalog" = 'SNOWFLAKE' THEN 'Snowflake-Managed'
        ELSE 'Externally-Managed'
    END AS management_type,
    COUNT(*) AS iceberg_count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "catalog"
ORDER BY iceberg_count DESC;

-- Iceberg tables by management type (summary)
SHOW ICEBERG TABLES IN ACCOUNT;
SELECT 
    CASE 
        WHEN "catalog" = 'SNOWFLAKE' THEN 'Snowflake-Managed'
        ELSE 'Externally-Managed (Catalog-Linked)'
    END AS management_type,
    COUNT(*) AS iceberg_count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY 1
ORDER BY iceberg_count DESC;

-- Iceberg tables by catalog type and database
SHOW ICEBERG TABLES IN ACCOUNT;
SELECT 
    "database_name",
    "catalog" AS catalog_type,
    CASE 
        WHEN "catalog" = 'SNOWFLAKE' THEN 'Snowflake-Managed'
        ELSE 'Externally-Managed'
    END AS management_type,
    COUNT(*) AS iceberg_count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "database_name", "catalog"
ORDER BY iceberg_count DESC;

-- Iceberg tables by catalog type, database and schema
SHOW ICEBERG TABLES IN ACCOUNT;
SELECT 
    "database_name",
    "schema_name",
    "catalog" AS catalog_type,
    CASE 
        WHEN "catalog" = 'SNOWFLAKE' THEN 'Snowflake-Managed'
        ELSE 'Externally-Managed'
    END AS management_type,
    COUNT(*) AS iceberg_count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "database_name", "schema_name", "catalog"
ORDER BY iceberg_count DESC;

-- Detailed Iceberg tables with catalog and external volume info
SHOW ICEBERG TABLES IN ACCOUNT;
SELECT 
    "database_name", 
    "schema_name", 
    "name" AS table_name,
    "catalog" AS catalog_type,
    CASE 
        WHEN "catalog" = 'SNOWFLAKE' THEN 'Snowflake-Managed'
        ELSE 'Externally-Managed'
    END AS management_type,
    "external_volume_name",
    "catalog_table_name",
    "base_location",
    "created_on"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
ORDER BY "database_name", "schema_name", "name";

-- Summary: Snowflake-Managed vs External by database
SHOW ICEBERG TABLES IN ACCOUNT;
SELECT 
    "database_name",
    SUM(CASE WHEN "catalog" = 'SNOWFLAKE' THEN 1 ELSE 0 END) AS snowflake_managed,
    SUM(CASE WHEN "catalog" != 'SNOWFLAKE' THEN 1 ELSE 0 END) AS externally_managed,
    COUNT(*) AS total_iceberg
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "database_name"
ORDER BY total_iceberg DESC;

-- All external catalog types in use
SHOW ICEBERG TABLES IN ACCOUNT;
SELECT DISTINCT 
    "catalog" AS catalog_type,
    CASE 
        WHEN "catalog" = 'SNOWFLAKE' THEN 'Snowflake-Managed Iceberg Tables'
        WHEN "catalog" = 'GLUE' THEN 'AWS Glue Data Catalog'
        WHEN "catalog" = 'POLARIS' THEN 'Snowflake Open Catalog (Polaris)'
        WHEN "catalog" = 'OBJECT_STORE' THEN 'Object Store (no catalog)'
        ELSE 'Other External Catalog'
    END AS catalog_description
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
ORDER BY "catalog";
```

### Iceberg Tables Storage and Details
```sql
-- Iceberg tables with storage info (from ACCOUNT_USAGE)
SELECT 
    table_catalog AS database_name,
    table_schema AS schema_name,
    table_name,
    table_type,
    created AS created_on,
    last_altered,
    bytes,
    row_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
  AND is_iceberg = 'YES'
ORDER BY table_catalog, table_schema, table_name
LIMIT 100;

-- Iceberg tables count by table type
SELECT 
    table_type,
    COUNT(*) AS iceberg_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
  AND is_iceberg = 'YES'
GROUP BY table_type
ORDER BY iceberg_count DESC;

-- Iceberg tables summary by database with storage info
SELECT 
    table_catalog AS database_name,
    COUNT(*) AS iceberg_count,
    SUM(bytes) / POWER(1024, 3) AS total_size_gb,
    SUM(row_count) AS total_rows
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
  AND is_iceberg = 'YES'
GROUP BY table_catalog
ORDER BY iceberg_count DESC;

-- Real-time Iceberg tables for specific database
-- Replace <DATABASE_NAME> with actual database name
SHOW ICEBERG TABLES IN DATABASE <DATABASE_NAME>;
SELECT "database_name", "schema_name", "name", "catalog", 
       "external_volume_name", "base_location", "created_on"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
ORDER BY "schema_name", "name";
```

### Object Types by Database/Schema
```sql
-- All object types count in account
SELECT 
    'Tables' AS object_type, COUNT(*) AS count FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND table_type = 'BASE TABLE'
UNION ALL SELECT 'Views', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND table_type = 'VIEW'
UNION ALL SELECT 'Materialized Views', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND table_type = 'MATERIALIZED VIEW'
UNION ALL SELECT 'External Tables', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND table_type = 'EXTERNAL TABLE'
UNION ALL SELECT 'Dynamic Tables', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND table_type = 'DYNAMIC TABLE'
UNION ALL SELECT 'Iceberg Tables', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND is_iceberg = 'YES'
UNION ALL SELECT 'Procedures', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.PROCEDURES WHERE deleted IS NULL
UNION ALL SELECT 'Functions', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.FUNCTIONS WHERE deleted IS NULL
UNION ALL SELECT 'Stages', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.STAGES WHERE deleted IS NULL
UNION ALL SELECT 'Streams', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.STREAMS WHERE deleted IS NULL
UNION ALL SELECT 'Tasks', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TASKS WHERE deleted IS NULL
UNION ALL SELECT 'Pipes', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.PIPES WHERE deleted IS NULL
UNION ALL SELECT 'Sequences', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.SEQUENCES WHERE deleted IS NULL
UNION ALL SELECT 'File Formats', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.FILE_FORMATS WHERE deleted IS NULL
ORDER BY count DESC;

-- Object types by database
SELECT 
    table_catalog AS database_name,
    SUM(CASE WHEN table_type = 'BASE TABLE' THEN 1 ELSE 0 END) AS tables,
    SUM(CASE WHEN table_type = 'VIEW' THEN 1 ELSE 0 END) AS views,
    SUM(CASE WHEN table_type = 'MATERIALIZED VIEW' THEN 1 ELSE 0 END) AS mat_views,
    SUM(CASE WHEN table_type = 'EXTERNAL TABLE' THEN 1 ELSE 0 END) AS ext_tables,
    SUM(CASE WHEN table_type = 'DYNAMIC TABLE' THEN 1 ELSE 0 END) AS dyn_tables,
    COUNT(*) AS total
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
GROUP BY table_catalog
ORDER BY total DESC
LIMIT 20;

-- Object types by schema (for specific database)
SELECT 
    table_schema AS schema_name,
    SUM(CASE WHEN table_type = 'BASE TABLE' THEN 1 ELSE 0 END) AS tables,
    SUM(CASE WHEN table_type = 'VIEW' THEN 1 ELSE 0 END) AS views,
    SUM(CASE WHEN table_type = 'MATERIALIZED VIEW' THEN 1 ELSE 0 END) AS mat_views,
    SUM(CASE WHEN table_type = 'EXTERNAL TABLE' THEN 1 ELSE 0 END) AS ext_tables,
    SUM(CASE WHEN table_type = 'DYNAMIC TABLE' THEN 1 ELSE 0 END) AS dyn_tables,
    COUNT(*) AS total
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
  AND table_catalog = '<DATABASE_NAME>'  -- Replace with your database
GROUP BY table_schema
ORDER BY total DESC;

-- Procedures by database
SELECT 
    procedure_catalog AS database_name,
    COUNT(*) AS procedure_count
FROM SNOWFLAKE.ACCOUNT_USAGE.PROCEDURES
WHERE deleted IS NULL
GROUP BY procedure_catalog
ORDER BY procedure_count DESC
LIMIT 20;

-- Functions by database
SELECT 
    function_catalog AS database_name,
    COUNT(*) AS function_count
FROM SNOWFLAKE.ACCOUNT_USAGE.FUNCTIONS
WHERE deleted IS NULL
GROUP BY function_catalog
ORDER BY function_count DESC
LIMIT 20;

-- Stages by database
SELECT 
    stage_catalog AS database_name,
    stage_type,
    COUNT(*) AS stage_count
FROM SNOWFLAKE.ACCOUNT_USAGE.STAGES
WHERE deleted IS NULL
GROUP BY stage_catalog, stage_type
ORDER BY stage_count DESC
LIMIT 20;

-- Streams by database
SELECT 
    table_catalog AS database_name,
    COUNT(*) AS stream_count
FROM SNOWFLAKE.ACCOUNT_USAGE.STREAMS
WHERE deleted IS NULL
GROUP BY table_catalog
ORDER BY stream_count DESC
LIMIT 20;

-- Tasks by database and state
SELECT 
    database_name,
    state,
    COUNT(*) AS task_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TASKS
WHERE deleted IS NULL
GROUP BY database_name, state
ORDER BY task_count DESC
LIMIT 20;

-- Pipes by database
SELECT 
    pipe_catalog AS database_name,
    COUNT(*) AS pipe_count
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPES
WHERE deleted IS NULL
GROUP BY pipe_catalog
ORDER BY pipe_count DESC
LIMIT 20;

-- Complete object inventory for a specific database (real-time)
-- Replace <DATABASE_NAME> with actual database name
SHOW TABLES IN DATABASE <DATABASE_NAME>;
SHOW VIEWS IN DATABASE <DATABASE_NAME>;
SHOW PROCEDURES IN DATABASE <DATABASE_NAME>;
SHOW USER FUNCTIONS IN DATABASE <DATABASE_NAME>;
SHOW STAGES IN DATABASE <DATABASE_NAME>;
SHOW STREAMS IN DATABASE <DATABASE_NAME>;
SHOW TASKS IN DATABASE <DATABASE_NAME>;
SHOW PIPES IN DATABASE <DATABASE_NAME>;
```

---

## Detailed Inventory Report

```sql
-- Complete inventory with breakdown
WITH inventory AS (
    SELECT 'Users (Active)' AS metric, COUNT(*) AS count FROM SNOWFLAKE.ACCOUNT_USAGE.USERS WHERE deleted_on IS NULL AND disabled = 'false'
    UNION ALL SELECT 'Users (Disabled)', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.USERS WHERE deleted_on IS NULL AND disabled = 'true'
    UNION ALL SELECT 'Databases', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES WHERE deleted IS NULL
    UNION ALL SELECT 'Warehouses', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES WHERE deleted IS NULL
    UNION ALL SELECT 'Account Roles', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES WHERE deleted_on IS NULL AND is_database_role = 'NO'
    UNION ALL SELECT 'Database Roles', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES WHERE deleted_on IS NULL AND is_database_role = 'YES'
    UNION ALL SELECT 'Schemas', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.SCHEMATA WHERE deleted IS NULL
    UNION ALL SELECT 'Tables', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND table_type = 'BASE TABLE'
    UNION ALL SELECT 'Views', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND table_type = 'VIEW'
    UNION ALL SELECT 'Materialized Views', COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES WHERE deleted IS NULL AND table_type = 'MATERIALIZED VIEW'
)
SELECT metric, count FROM inventory ORDER BY metric;
```

---

## Real-Time Counts (Using SHOW Commands)

```sql
-- For immediate results without ACCOUNT_USAGE latency
-- Run each separately

SHOW USERS;
-- Then: SELECT COUNT(*) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

SHOW DATABASES;
-- Then: SELECT COUNT(*) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

SHOW WAREHOUSES;
-- Then: SELECT COUNT(*) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

SHOW ROLES;
-- Then: SELECT COUNT(*) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

SHOW MANAGED ACCOUNTS;
-- Then: SELECT COUNT(*) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

SHOW RESOURCE MONITORS;
-- Then: SELECT COUNT(*) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

SHOW NETWORK POLICIES;
-- Then: SELECT COUNT(*) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

---

## Required Privileges

```
ACCOUNT_USAGE views require:
  GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <role_name>;

SHOW commands require:
  Various privileges depending on object type
  ACCOUNTADMIN can see all objects
```

---

## Notes
- ACCOUNT_USAGE has ~45 minute latency
- Use SHOW commands for real-time counts
- Counts exclude deleted objects (deleted IS NULL filter)
