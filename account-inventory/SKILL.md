---
name: account-inventory-local
description: |
  Snowflake account inventory and statistics for administration, audits, and capacity planning.

  Topics:
    Users - users, user count, accounts, service accounts, person users, user type, legacy service
    Databases - databases, db count, untagged databases, tagged databases
    Warehouses - warehouses, compute, wh count, gen1, gen2, generation, snowpark, snowpark-optimized, warehouse tag, tagged warehouses, untagged warehouses
    Roles - roles, role count, RBAC
    Schemas - schemas, schema count
    Tables - tables, table count, objects
    Reader Accounts - reader, reader accounts, managed accounts
    Shares - shares, inbound shares, outbound shares, data sharing, share count, internal sharing, external sharing, marketplace listing, direct share, organization sharing
    Resource Monitors - resource monitor, monitors, credit quota
    Network Policies - network policy, network policies, IP whitelist, allowed IPs
    Integrations - integrations, connectors, storage integration, API integration, catalog integration, notification, external access, SCIM, OAuth
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

-- Databases count by tag name
SELECT 
    tr.TAG_DATABASE || '.' || tr.TAG_SCHEMA || '.' || tr.TAG_NAME AS full_tag_name,
    COUNT(DISTINCT tr.OBJECT_NAME) AS database_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
WHERE tr.DOMAIN = 'DATABASE'
  AND tr.OBJECT_DELETED IS NULL
GROUP BY full_tag_name
ORDER BY database_count DESC;

-- Databases count by tag name and value
SELECT 
    tr.TAG_DATABASE || '.' || tr.TAG_SCHEMA || '.' || tr.TAG_NAME AS full_tag_name,
    tr.TAG_VALUE,
    COUNT(DISTINCT tr.OBJECT_NAME) AS database_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
WHERE tr.DOMAIN = 'DATABASE'
  AND tr.OBJECT_DELETED IS NULL
GROUP BY full_tag_name, tr.TAG_VALUE
ORDER BY full_tag_name, database_count DESC;

-- Tagged vs untagged database count
WITH all_databases AS (
    SELECT DATABASE_NAME 
    FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES 
    WHERE deleted IS NULL
),
tagged_databases AS (
    SELECT DISTINCT OBJECT_NAME AS database_name
    FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
    WHERE DOMAIN = 'DATABASE'
      AND OBJECT_DELETED IS NULL
)
SELECT 
    CASE WHEN td.database_name IS NOT NULL THEN 'Tagged' ELSE 'Untagged' END AS status,
    COUNT(*) AS database_count
FROM all_databases ad
LEFT JOIN tagged_databases td ON ad.DATABASE_NAME = td.database_name
GROUP BY 1
ORDER BY 1;

-- List untagged databases
WITH tagged_databases AS (
    SELECT DISTINCT OBJECT_NAME AS database_name
    FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
    WHERE DOMAIN = 'DATABASE'
      AND OBJECT_DELETED IS NULL
)
SELECT 
    d.DATABASE_NAME,
    d.DATABASE_OWNER,
    d.CREATED,
    d.LAST_ALTERED
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES d
WHERE d.deleted IS NULL
  AND d.DATABASE_NAME NOT IN (SELECT database_name FROM tagged_databases)
ORDER BY d.DATABASE_NAME;

-- List untagged databases (excluding sandbox/dev patterns)
WITH tagged_databases AS (
    SELECT DISTINCT OBJECT_NAME AS database_name
    FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
    WHERE DOMAIN = 'DATABASE'
      AND OBJECT_DELETED IS NULL
)
SELECT 
    d.DATABASE_NAME,
    d.DATABASE_OWNER,
    d.CREATED,
    d.LAST_ALTERED
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES d
WHERE d.deleted IS NULL
  AND d.DATABASE_NAME NOT IN (SELECT database_name FROM tagged_databases)
  AND d.DATABASE_NAME NOT ILIKE '%SANDBOX%'
  AND d.DATABASE_NAME NOT ILIKE '%_DEV'
  AND d.DATABASE_NAME NOT ILIKE 'DEV_%'
  AND d.DATABASE_NAME NOT ILIKE '%_TEST'
  AND d.DATABASE_NAME NOT ILIKE 'TEST_%'
ORDER BY d.DATABASE_NAME;

-- List databases with their tags
SELECT 
    tr.OBJECT_NAME AS database_name,
    tr.TAG_DATABASE || '.' || tr.TAG_SCHEMA || '.' || tr.TAG_NAME AS full_tag_name,
    tr.TAG_VALUE
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
WHERE tr.DOMAIN = 'DATABASE'
  AND tr.OBJECT_DELETED IS NULL
ORDER BY tr.OBJECT_NAME, full_tag_name;
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

-- Total warehouse count (real-time)
SHOW WAREHOUSES;
SELECT COUNT(*) AS warehouse_count FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Warehouses by size
SHOW WAREHOUSES;
SELECT 
    "size" AS warehouse_size,
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "size"
ORDER BY count DESC;

-- Warehouses by generation (Gen1 vs Gen2)
-- Gen2 warehouses have improved performance and efficiency
-- Legacy warehouses (NULL generation) are older warehouses before generations existed
SHOW WAREHOUSES;
SELECT 
    COALESCE("generation"::VARCHAR, 'Legacy (None)') AS generation,
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "generation"
ORDER BY count DESC;

-- Warehouses by type (Standard vs Snowpark-Optimized)
-- SNOWPARK-OPTIMIZED warehouses have more memory per node for ML/data science workloads
SHOW WAREHOUSES;
SELECT 
    "type" AS warehouse_type,
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "type"
ORDER BY count DESC;

-- Warehouses by generation and type combined
SHOW WAREHOUSES;
SELECT 
    COALESCE("generation"::VARCHAR, 'Legacy (None)') AS generation,
    "type" AS warehouse_type,
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "generation", "type"
ORDER BY generation, warehouse_type;

-- Warehouses by size and generation
SHOW WAREHOUSES;
SELECT 
    "size" AS warehouse_size,
    COALESCE("generation"::VARCHAR, 'Legacy') AS generation,
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "size", "generation"
ORDER BY "size", generation;

-- Detailed warehouse list with generation and type
SHOW WAREHOUSES;
SELECT 
    "name" AS warehouse_name,
    "size" AS warehouse_size,
    "type" AS warehouse_type,
    COALESCE("generation"::VARCHAR, 'Legacy (None)') AS generation,
    "state",
    "auto_suspend",
    "auto_resume",
    "owner"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
ORDER BY "name";

-- Snowpark-optimized warehouses only
SHOW WAREHOUSES;
SELECT 
    "name" AS warehouse_name,
    "size" AS warehouse_size,
    COALESCE("generation"::VARCHAR, 'Legacy') AS generation,
    "state",
    "owner"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "type" = 'SNOWPARK-OPTIMIZED'
ORDER BY "name";

-- Gen2 warehouses only
SHOW WAREHOUSES;
SELECT 
    "name" AS warehouse_name,
    "size" AS warehouse_size,
    "type" AS warehouse_type,
    "state",
    "owner"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "generation" = 2
ORDER BY "name";

-- Gen1 warehouses (candidates for upgrade to Gen2)
SHOW WAREHOUSES;
SELECT 
    "name" AS warehouse_name,
    "size" AS warehouse_size,
    "type" AS warehouse_type,
    "state",
    "owner"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "generation" = 1
ORDER BY "name";

-- Warehouses count by tag name
SELECT 
    tr.TAG_DATABASE || '.' || tr.TAG_SCHEMA || '.' || tr.TAG_NAME AS full_tag_name,
    COUNT(DISTINCT tr.OBJECT_NAME) AS warehouse_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
WHERE tr.DOMAIN = 'WAREHOUSE'
  AND tr.OBJECT_DELETED IS NULL
GROUP BY full_tag_name
ORDER BY warehouse_count DESC;

-- Warehouses count by tag name and value
SELECT 
    tr.TAG_DATABASE || '.' || tr.TAG_SCHEMA || '.' || tr.TAG_NAME AS full_tag_name,
    tr.TAG_VALUE,
    COUNT(DISTINCT tr.OBJECT_NAME) AS warehouse_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
WHERE tr.DOMAIN = 'WAREHOUSE'
  AND tr.OBJECT_DELETED IS NULL
GROUP BY full_tag_name, tr.TAG_VALUE
ORDER BY full_tag_name, warehouse_count DESC;

-- Tagged vs untagged warehouse count (real-time)
WITH all_warehouses AS (
    SELECT "name" AS warehouse_name FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
),
tagged_warehouses AS (
    SELECT DISTINCT OBJECT_NAME AS warehouse_name
    FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
    WHERE DOMAIN = 'WAREHOUSE'
      AND OBJECT_DELETED IS NULL
)
SELECT 
    CASE WHEN tw.warehouse_name IS NOT NULL THEN 'Tagged' ELSE 'Untagged' END AS status,
    COUNT(*) AS warehouse_count
FROM (SHOW WAREHOUSES) aw
LEFT JOIN tagged_warehouses tw ON aw."name" = tw.warehouse_name
GROUP BY 1
ORDER BY 1;

-- List untagged warehouses
WITH tagged_warehouses AS (
    SELECT DISTINCT OBJECT_NAME AS warehouse_name
    FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
    WHERE DOMAIN = 'WAREHOUSE'
      AND OBJECT_DELETED IS NULL
)
SELECT 
    w."name" AS warehouse_name,
    w."size" AS warehouse_size,
    w."type" AS warehouse_type,
    w."owner" AS owner
FROM (SHOW WAREHOUSES) w
WHERE w."name" NOT IN (SELECT warehouse_name FROM tagged_warehouses)
ORDER BY w."name";

-- List warehouses with their tags
SELECT 
    tr.OBJECT_NAME AS warehouse_name,
    tr.TAG_DATABASE || '.' || tr.TAG_SCHEMA || '.' || tr.TAG_NAME AS full_tag_name,
    tr.TAG_VALUE
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
WHERE tr.DOMAIN = 'WAREHOUSE'
  AND tr.OBJECT_DELETED IS NULL
ORDER BY tr.OBJECT_NAME, full_tag_name;
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

-- Internal vs External sharing (by organization)
-- Internal = same organization (RJXZPFF.*), External = other organizations
SHOW SHARES;
SELECT 
    "kind",
    CASE 
        WHEN SPLIT_PART("owner_account", '.', 1) = SPLIT_PART(CURRENT_ACCOUNT(), '.', 1) 
        THEN 'Internal (Same Org)'
        ELSE 'External (Other Org)'
    END AS sharing_type,
    COUNT(*) AS share_count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "kind", sharing_type
ORDER BY "kind", sharing_type;

-- Internal shares summary (within your organization)
SHOW SHARES;
SELECT 
    "kind",
    "owner_account",
    COUNT(*) AS share_count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE SPLIT_PART("owner_account", '.', 1) = SPLIT_PART(CURRENT_ACCOUNT(), '.', 1)
GROUP BY "kind", "owner_account"
ORDER BY share_count DESC;

-- External shares summary (from/to other organizations)
SHOW SHARES;
SELECT 
    "kind",
    "owner_account",
    COUNT(*) AS share_count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE SPLIT_PART("owner_account", '.', 1) != SPLIT_PART(CURRENT_ACCOUNT(), '.', 1)
GROUP BY "kind", "owner_account"
ORDER BY share_count DESC;

-- Inbound shares by source organization
SHOW SHARES;
SELECT 
    SPLIT_PART("owner_account", '.', 1) AS source_org,
    "owner_account" AS source_account,
    COUNT(*) AS inbound_shares
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "kind" = 'INBOUND'
GROUP BY source_org, source_account
ORDER BY inbound_shares DESC;

-- Outbound shares with consumer count
SHOW SHARES;
SELECT 
    "name" AS share_name,
    "database_name",
    CASE 
        WHEN "to" IS NULL OR "to" = '' THEN 0
        WHEN "to" = '''TARGETED ACCOUNTS''' THEN -1  -- Marketplace listing
        ELSE ARRAY_SIZE(SPLIT("to", ','))
    END AS consumer_count,
    CASE 
        WHEN "listing_global_name" IS NOT NULL AND "listing_global_name" != '' 
        THEN 'Marketplace Listing'
        WHEN "to" IS NULL OR "to" = '' THEN 'No Consumers'
        ELSE 'Direct Share'
    END AS share_method
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "kind" = 'OUTBOUND'
ORDER BY consumer_count DESC;

-- Marketplace listings vs direct shares
SHOW SHARES;
SELECT 
    CASE 
        WHEN "listing_global_name" IS NOT NULL AND "listing_global_name" != '' 
        THEN 'Marketplace Listing'
        ELSE 'Direct Share'
    END AS share_method,
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "kind" = 'OUTBOUND'
GROUP BY share_method
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

### Integrations Count (Connectors)
```sql
-- Total integrations count
SHOW INTEGRATIONS;
SELECT COUNT(*) AS integration_count FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Integrations by type and category
SHOW INTEGRATIONS;
SELECT 
    "type" AS integration_type,
    "category",
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "type", "category"
ORDER BY count DESC;

-- Integrations by category (summary)
SHOW INTEGRATIONS;
SELECT 
    "category",
    COUNT(*) AS count
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
GROUP BY "category"
ORDER BY count DESC;

-- Storage integrations (S3, Azure, GCS)
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "enabled",
    "created_on",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "category" = 'STORAGE'
ORDER BY "created_on" DESC;

-- Catalog integrations (Glue, Polaris, Object Store)
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "enabled",
    "created_on",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "category" = 'CATALOG'
ORDER BY "created_on" DESC;

-- API integrations (external functions)
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "enabled",
    "created_on",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "category" = 'API'
ORDER BY "created_on" DESC;

-- Notification integrations (email, SNS, webhook)
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "enabled",
    "created_on",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "category" = 'NOTIFICATION'
ORDER BY "type", "created_on" DESC;

-- Security integrations (SCIM, OAuth)
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "enabled",
    "created_on",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "category" = 'SECURITY'
ORDER BY "created_on" DESC;

-- External access integrations
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "enabled",
    "created_on",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "category" = 'EXTERNAL_ACCESS'
ORDER BY "created_on" DESC;

-- Disabled integrations
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "category",
    "created_on",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE "enabled" = 'false'
ORDER BY "category", "name";

-- Integrations with details
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "category",
    "enabled",
    "created_on",
    "comment"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
ORDER BY "category", "type", "name";

-- Recent integrations (last 30 days)
SHOW INTEGRATIONS;
SELECT 
    "name",
    "type",
    "category",
    "enabled",
    "created_on"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
WHERE TRY_TO_TIMESTAMP("created_on") > DATEADD(day, -30, CURRENT_TIMESTAMP())
ORDER BY "created_on" DESC;
```

### Tags Count
```sql
-- Total tags count
SELECT COUNT(*) AS tag_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAGS
WHERE deleted IS NULL;

-- Tags by database
SELECT 
    tag_database AS database_name,
    COUNT(*) AS tag_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAGS
WHERE deleted IS NULL
GROUP BY tag_database
ORDER BY tag_count DESC;

-- Tags by database and schema
SELECT 
    tag_database AS database_name,
    tag_schema AS schema_name,
    COUNT(*) AS tag_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAGS
WHERE deleted IS NULL
GROUP BY tag_database, tag_schema
ORDER BY tag_count DESC;

-- Tags with details (name, allowed values, owner)
SELECT 
    tag_database,
    tag_schema,
    tag_name,
    tag_owner,
    allowed_values,
    created,
    last_altered
FROM SNOWFLAKE.ACCOUNT_USAGE.TAGS
WHERE deleted IS NULL
ORDER BY tag_database, tag_schema, tag_name;

-- Tags with allowed values defined
SELECT 
    tag_database || '.' || tag_schema || '.' || tag_name AS fully_qualified_name,
    allowed_values,
    tag_owner
FROM SNOWFLAKE.ACCOUNT_USAGE.TAGS
WHERE deleted IS NULL
  AND allowed_values IS NOT NULL
ORDER BY fully_qualified_name;

-- Real-time tags count
SHOW TAGS IN ACCOUNT;
SELECT COUNT(*) AS tag_count FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Real-time tags with details
SHOW TAGS IN ACCOUNT;
SELECT "database_name", "schema_name", "name", "owner", "allowed_values", "created_on"
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
ORDER BY "database_name", "schema_name", "name";
```

### Tag References (Tagged Objects)
```sql
-- Total tag references count (objects with tags)
SELECT COUNT(*) AS tag_reference_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL;

-- Tag references by object type
SELECT 
    domain AS object_type,
    COUNT(*) AS reference_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
GROUP BY domain
ORDER BY reference_count DESC;

-- Tag references by tag name
SELECT 
    tag_database || '.' || tag_schema || '.' || tag_name AS tag_fqn,
    COUNT(*) AS usage_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
GROUP BY tag_database, tag_schema, tag_name
ORDER BY usage_count DESC
LIMIT 20;

-- Tag references by database
SELECT 
    object_database AS database_name,
    COUNT(*) AS tagged_objects
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
  AND object_database IS NOT NULL
GROUP BY object_database
ORDER BY tagged_objects DESC;

-- Tag value distribution (most common tag values)
SELECT 
    tag_database || '.' || tag_schema || '.' || tag_name AS tag_fqn,
    tag_value,
    COUNT(*) AS usage_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
GROUP BY tag_database, tag_schema, tag_name, tag_value
ORDER BY usage_count DESC
LIMIT 50;

-- Tagged tables
SELECT 
    object_database || '.' || object_schema || '.' || object_name AS table_fqn,
    tag_database || '.' || tag_schema || '.' || tag_name AS tag_fqn,
    tag_value
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
  AND domain = 'TABLE'
ORDER BY object_database, object_schema, object_name
LIMIT 100;

-- Tagged columns (useful for sensitive data tracking)
SELECT 
    object_database || '.' || object_schema || '.' || object_name AS table_fqn,
    column_name,
    tag_database || '.' || tag_schema || '.' || tag_name AS tag_fqn,
    tag_value
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
  AND domain = 'COLUMN'
ORDER BY object_database, object_schema, object_name, column_name
LIMIT 100;

-- Snowflake system tags (data classification)
SELECT 
    tag_name,
    tag_value,
    COUNT(*) AS column_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
  AND tag_database = 'SNOWFLAKE'
  AND tag_schema = 'CORE'
GROUP BY tag_name, tag_value
ORDER BY tag_name, column_count DESC;

-- PII/Sensitive data tagged columns (system classification)
SELECT 
    object_database || '.' || object_schema || '.' || object_name AS table_fqn,
    column_name,
    tag_name,
    tag_value AS classification
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
  AND domain = 'COLUMN'
  AND tag_database = 'SNOWFLAKE'
  AND tag_schema = 'CORE'
  AND tag_name IN ('SEMANTIC_CATEGORY', 'PRIVACY_CATEGORY')
ORDER BY object_database, object_schema, object_name, column_name
LIMIT 100;

-- Objects by tag (find all objects with specific tag)
-- Replace <TAG_DATABASE>, <TAG_SCHEMA>, <TAG_NAME> with actual values
SELECT 
    domain AS object_type,
    object_database,
    object_schema,
    object_name,
    column_name,
    tag_value
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
  AND tag_database = '<TAG_DATABASE>'
  AND tag_schema = '<TAG_SCHEMA>'
  AND tag_name = '<TAG_NAME>'
ORDER BY domain, object_database, object_schema, object_name;

-- Untagged tables (tables without any tags)
SELECT t.table_catalog, t.table_schema, t.table_name
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
    ON t.table_catalog = tr.object_database
    AND t.table_schema = tr.object_schema
    AND t.table_name = tr.object_name
    AND tr.domain = 'TABLE'
    AND tr.object_deleted IS NULL
WHERE t.deleted IS NULL
  AND t.table_type = 'BASE TABLE'
  AND tr.tag_name IS NULL
ORDER BY t.table_catalog, t.table_schema, t.table_name
LIMIT 100;

-- Tag summary by classification category
SELECT 
    CASE 
        WHEN tag_name = 'PRIVACY_CATEGORY' THEN 'Privacy'
        WHEN tag_name = 'SEMANTIC_CATEGORY' THEN 'Semantic'
        ELSE 'Custom'
    END AS category_type,
    tag_value,
    COUNT(*) AS count
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE object_deleted IS NULL
  AND domain = 'COLUMN'
GROUP BY category_type, tag_value
ORDER BY category_type, count DESC;
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
