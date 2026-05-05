# Lake Formation Transition Guide — Single Account Migrations

## Overview

AWS Lake Formation can be used at any time to manage metadata (databases, tables, columns) without any impact to existing users. Granting Lake Formation permissions on catalog resources is a no-op until IAMAllowedPrincipals is removed and S3 locations are registered for credential vending. This means you can begin setting up Lake Formation permissions incrementally, well before "flipping the switch."

For storage-level permissions, S3 bucket policies continue to apply regardless of Lake Formation configuration. Lake Formation credential vending controls access to data through temporary credentials — it does not modify or replace your S3 bucket policies.

This guide covers two approaches to transitioning a single AWS account to Lake Formation credential vending. Both are designed to be incremental and reversible.

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
