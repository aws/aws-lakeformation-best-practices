# Lake Formation Transition Guide — Single Account Migrations

## Overview

Many organizations start their data lake journey with IAM policies and S3 bucket policies controlling access to data. As the environment grows — more tables, more consumers, more teams — managing access through IAM alone becomes increasingly complex and error-prone. AWS Lake Formation simplifies this by providing a centralized permissions model that sits on top of the AWS Glue Data Catalog: you grant access to databases, tables, and columns directly, and Lake Formation vends temporary credentials to authorized principals.

The challenge is not adopting Lake Formation — it's transitioning to it without disrupting existing workloads. A production environment with active consumers cannot simply "flip a switch." The transition needs to be incremental, reversible, and low-risk.

### What You Should Know Before Starting

* Lake Formation metadata permissions are safe to set up at any time. Granting LF permissions on databases, tables, and columns is a no-op until IAMAllowedPrincipals is removed and S3 locations are registered for credential vending. You can begin layering in LF permissions well before "flipping the switch."
* S3 bucket policies still apply. Lake Formation credential vending controls access to data through temporary credentials — it does not modify or replace your S3 bucket policies.
* The transition is incremental. You do not need to migrate everything at once. Both plans in this guide are designed for step-by-step migration with validation at each stage.

### What This Guide Covers

This guide presents two approaches to transitioning a single AWS account to Lake Formation credential vending. Plan #1 (Per Resource) migrates table by table — all consumers of a resource move together. Plan #2 (Per User) migrates user by user — individual principals are opted in while others remain on IAM. Both approaches are reversible, and guidance is provided on when to use each.

---

## Prerequisites (Both Plans)

Before starting either transition plan:

1. **Designate a Lake Formation Administrator** — At minimum, one IAM principal must be registered as an LF Admin. Optionally, designate database-level administrators for delegation.
2. **Add `lakeformation:GetDataAccess` to all consumer IAM policies** — This permission is required for credential vending. It is a **no-op** when no locations are registered, so it is safe (and recommended) to add upfront for all principals. This eliminates a class of errors during migration.
3. **Enable CloudTrail for Glue Data Catalog events** — Management events (on by default) capture `GetTable`, `GetPartitions`, `BatchGetPartition`, `CreateTable`, etc. These are essential for identifying consumers.
4. **Identify service roles** — Services like Redshift Spectrum, EMR, and Athena use service/execution roles. These roles are treated the same as user roles — they need LF grants and `lakeformation:GetDataAccess`.
5. **Decide on a permissions model** — Choose between named-resource permissions or LF-Tags (tag-based access control). This decision affects how you grant permissions but does not change the migration mechanics.

---

## Transition Plans

### Plan #1 — Per Resource Transition

Migrate **table by table** (or location by location). For each resource, all consumers are migrated at once. This is a clean "all or nothing" approach per resource.

- Register S3 locations with full registration
- Grant LF permissions to all consumers of a table before activating credential vending
- Rollback by deregistering the location (affects all consumers of that table)

👉 [Full details: Transition Plan #1 — Per Resource](transition-guide/single-account-transition-plan-1-per-resource.md)

### Plan #2 — Per User Transition

Migrate **user by user** (or cohort by cohort). Hybrid access mode allows you to opt individual principals into Lake Formation credential vending while others continue using IAM.

- Register S3 locations in Hybrid mode
- Use `CreateLakeFormationOptIn` to flip individual users to LF
- Rollback by removing the opt-in for a specific user (surgical, no impact to others)

👉 [Full details: Transition Plan #2 — Per User](transition-guide/single-account-transition-plan-2-per-user.md)

---

## When to Use Plan #1 vs Plan #2

| | **Plan #1 — Per Resource** | **Plan #2 — Per User** |
|---|---|---|
| **Best for** | Environments with many tables but few consumers per table; simpler IAM structures | Environments with many users accessing overlapping tables; complex orgs needing controlled rollout |
| **Pros** | Simpler — no Hybrid mode needed; clean "all or nothing" per table; easier to reason about completion | Granular control — migrate users at their own pace; lower risk per change; can pilot with a single team; allows rollback per user (remove opt-in) |
| **Cons** | If a table has many consumers, all must be ready simultaneously; harder to do partial rollback (deregister affects everyone); blocked by the "busiest" consumer | More complex orchestration; Hybrid mode + opt-in tracking required; must handle "shared table" conflicts (two users in different cohorts accessing same table); longer tail to full completion |
| **Rollback** | Deregister location (affects all consumers of that table) | Remove opt-in for specific user (surgical) |
| **Prerequisite complexity** | Must identify all consumers per table upfront | Must identify all tables per user upfront |
| **When it breaks down** | Tables with hundreds of consumers across many teams | Users who access hundreds of tables across many databases |

> **Tip:** Many real-world migrations combine both approaches — use Plan #2 for complex, high-traffic tables (migrate users incrementally) and Plan #1 for the long tail of rarely-used tables (migrate them in bulk).

---

## How Do You Find Out Who Is Consuming What?

Two complementary approaches:

### 1. CloudTrail Management Events (Recommended Starting Point)

Query CloudTrail for Glue Data Catalog API calls:
- `GetTable`, `GetTables`
- `GetPartitions`, `BatchGetPartition`
- `GetDatabase`
- `CreateTable`, `UpdateTable` (for identifying writers)

For each table, extract:
- The list of principals (IAM roles/users) accessing it
- Access frequency (helps prioritize)
- The table's S3 location

This tells you **who is actively accessing** each table.

??? note "Click to expand sample Athena Query for Lake Formation permissions from CloudTrail"
    ```sql
        WITH cloudtrail as (
            SELECT *,
                CASE
                    WHEN eventname in ('CreateDatabase', 'GetDatabases') THEN 'CATALOG'
                    WHEN eventname in (
                        'GetDatabase',
                        'UpdateDatabase',
                        'DeleteDatabase',
                        'CreateTable',
                        'GetTables'
                    ) THEN 'DATABASE'
                    WHEN eventname in (
                        'GetTable',
                        'GetTablesVersion',
                        'GetTablesVersions',
                        'GetPartition',
                        'GetUnfilteredPartition',
                        'GetInternalUnfilteredPartition',
                        'GetInternalUnfilteredPartitions',
                        'GetPartitions',
                        'GetUnfilteredPartitions',
                        'BatchGetPartition',
                        'GetPartitionIndexes',
                        'DESCRIBE',
                        'UpdateTable',
                        'DeleteTableVersion',
                        'BatchDeleteTableVersion',
                        'BatchCreatePartition',
                        'CreatePartition',
                        'DeletePartition',
                        'BatchDeletePartition',
                        'UpdatePartition',
                        'BatchUpdatePartition',
                        'CreatePartitionIndex',
                        'DeletePartitionIndex',
                        'DeleteTable'
                    ) THEN 'TABLE'
                END as resource_level
            FROM <INSERT CLOUDTRAIL TABLE HERE>
            WHERE eventsource = 'glue.amazonaws.com'
                and eventname in (
                    'GetDatabase',
                    'GetDatabases',
                    'UpdateDatabase',
                    'DeleteDatabase',
                    'CreateTable',
                    'CreateDatabase',
                    'GetTable',
                    'GetTables',
                    'GetTablesVersion',
                    'GetTablesVersions',
                    'GetPartition',
                    'GetUnfilteredPartition',
                    'GetInternalUnfilteredPartition',
                    'GetInternalUnfilteredPartitions',
                    'GetPartitions',
                    'GetUnfilteredPartitions',
                    'BatchGetPartition',
                    'GetPartitionIndexes',
                    'UpdateTable',
                    'DeleteTableVersion',
                    'BatchDeleteTableVersion',
                    'BatchCreatePartition',
                    'CreatePartition',
                    'DeletePartition',
                    'BatchDeletePartition',
                    'UpdatePartition',
                    'BatchUpdatePartition',
                    'CreatePartitionIndex',
                    'DeletePartitionIndex',
                    'DeleteTable'
                )
                and errorcode IS NULL -- We only support these useridentity types for now
                and useridentity.type in ('IAMUser', 'AssumedRole')
        )
        SELECT CASE
                WHEN useridentity.type = 'IAMUser' THEN useridentity.arn
                WHEN useridentity.type = 'AssumedRole' THEN useridentity.sessioncontext.sessionissuer.arn ELSE NULL
            END as user_arn,
            eventname,
            -- the following translation is not used, rather we use GlueDataCatalogActionTranslator instead.
            CASE
                -- Database level permissions
                WHEN eventname in ('GetDatabase') THEN 'DESCRIBE'
                WHEN eventname in ('UpdateDatabase') THEN 'ALTER'
                WHEN eventname in ('DeleteDatabase') THEN 'DROP'
                WHEN eventname in ('CreateTable') THEN 'CREATE_TABLE'
                WHEN eventname in ('CreateDatabase') THEN 'CREATE_DATABASE' -- These action have no permission requirements.
                -- WHEN eventname in ('GetDatabases') THEN 'LIST_DBS'
                WHEN eventname in ('GetTables') THEN 'DESCRIBE' -- Table Level permissions
                WHEN eventname in (
                    'GetTable',
                    'GetTablesVersion',
                    'GetTablesVersions',
                    'GetPartition',
                    'GetUnfilteredPartition',
                    'GetInternalUnfilteredPartition',
                    'GetInternalUnfilteredPartitions',
                    'GetPartitions',
                    'GetUnfilteredPartitions',
                    'BatchGetPartition',
                    'GetPartitionIndexes'
                ) THEN 'DESCRIBE'
                WHEN eventname in (
                    'UpdateTable',
                    'DeleteTableVersion',
                    'BatchDeleteTableVersion',
                    'BatchCreatePartition',
                    'CreatePartition',
                    'DeletePartition',
                    'BatchDeletePartition',
                    'UpdatePartition',
                    'BatchUpdatePartition',
                    'CreatePartitionIndex',
                    'DeletePartitionIndex'
                ) THEN 'ALTER'
                WHEN eventname in ('DeleteTable') THEN 'DROP' ELSE 'UNKNOWN'
            END as permission,
            resource_level,
            awsRegion,
            CASE
                WHEN json_extract_scalar(requestparameters, '$.catalogId') IS NOT NULL THEN json_extract_scalar(requestparameters, '$.catalogId') ELSE useridentity.accountid
            END as aws_account_id,
            CASE
                WHEN resource_level = 'DATABASE' THEN CASE
                    WHEN eventname in ('GetTables', 'CreateTable') THEN json_extract_scalar(requestparameters, '$.databaseName') ELSE json_extract_scalar(requestparameters, '$.name')
                END
                WHEN eventname = 'CreateDatabase' THEN json_extract_scalar(requestparameters, '$.databaseInput.name') ELSE json_extract_scalar(requestparameters, '$.databaseName')
            END as database_name,
            CASE
                WHEN resource_level = 'TABLE' THEN CASE
                    WHEN json_extract_scalar(requestparameters, '$.tableName') IS NULL THEN json_extract_scalar(requestparameters, '$.name') ELSE json_extract_scalar(requestparameters, '$.tableName')
                END
                WHEN eventname = 'CreateTable' THEN json_extract_scalar(requestparameters, '$.tableInput.name')
            END as table_name,
            SUM(1) as count
        FROM cloudtrail
        GROUP BY 1,2,3,4,5,6,7,8
    ```


??? note "Click to expand sample code to extract table locations"
    ```python
    import boto3
    import csv
    import sys

    glue = boto3.client('glue')

    writer = csv.writer(sys.stdout)
    writer.writerow(['catalog_id', 'database_name', 'table_name', 's3_location'])

    paginator = glue.get_paginator('get_databases')
    for db_page in paginator.paginate():
        for db in db_page['DatabaseList']:
            db_name = db['Name']
            catalog_id = db.get('CatalogId', '')

            table_paginator = glue.get_paginator('get_tables')
            for table_page in table_paginator.paginate(DatabaseName=db_name):
                for table in table_page['TableList']:
                    table_name = table['Name']
                    location = table.get('StorageDescriptor', {}).get('Location', '')
                    writer.writerow([catalog_id, db_name, table_name, location])
    ```

### 2. Policy Analysis

Parse existing IAM and resource policies:
- S3 bucket policies
- IAM role/user policies
- Glue Resource Policies

This tells you **who *could* access** the data (may be broader than active users — includes dormant permissions).

**Tool:** Use [policy-migrator-for-lake-formation](https://github.com/awslabs/policy-migrator-for-lake-formation) in **dry-run mode** to generate LF grants based on existing policies. This is complementary to CloudTrail analysis — it catches permissions that aren't actively exercised. 

---

## Best Practices & Considerations

### Batch by S3 Location, Not Just by Table

Multiple tables can share an S3 prefix. When you register a location, **all tables under that path** get credential vending at once. Always check what other tables exist under a path before registering — you may need to prepare grants for tables you hadn't planned to migrate yet.

### Add `lakeformation:GetDataAccess` Upfront

This is the single most effective thing you can do before starting either plan. It's a no-op without registered locations, so there is zero risk. Adding it to all principals at the start eliminates the most common migration error: "user can't access data because they're missing the IAM permission for credential vending."

### Service Roles Are Principals Too

Services like Redshift Spectrum, EMR, Athena, and Glue ETL use execution/service roles. These roles need:
- LF grants (just like user roles)
- `lakeformation:GetDataAccess` in their IAM policy
- Data Location Permissions (if they write data)

Don't forget to include these in your consumer analysis.

### Tracking Progress (Plan #2)

As you scale to many users and tables, maintain a manifest tracking:
- Which locations are registered (and in what mode)
- Which users are opted in for which resources
- Use `ListLakeFormationOptIns` API to audit progress

### New Users During Migration (Plan #2)

For locations already in Hybrid mode, **opt in new users immediately** rather than letting them start on IAM. This prevents growing the migration backlog and ensures new users are already on the target state.

---

## Completion Checklist

Once migration is complete (either plan), verify:

- [ ] All S3 data locations are registered (full registration, not Hybrid)
- [ ] IAMAllowedPrincipals removed from **all** databases and tables
- [ ] Direct S3 data access removed from consumer IAM policies (for migrated paths)
- [ ] Data Location Permissions granted for all write/create-table locations
- [ ] No Glue Resource Policies granting open catalog access remain
- [ ] All service roles (EMR, Redshift, Athena, Glue ETL) have LF grants
- [ ] `lakeformation:GetDataAccess` confirmed on all consumer IAM policies
- [ ] CloudTrail shows no access failures for a defined bake period (7–14 days recommended)
