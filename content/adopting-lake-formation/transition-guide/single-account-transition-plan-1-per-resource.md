# Transition Plan #1 — Per Resource Transition

In this approach, you migrate **table by table** (or location by location). For each resource, all consumers are migrated at once. This is a clean "all or nothing" approach per resource.

## Flow Diagram

??? note "Click to expand flow diagram"

    ```mermaid
    flowchart TD
        A[Identify consumers per table via CloudTrail] --> B[Prioritize tables: fewest consumers first]
        B --> C[Select next table/location to migrate]
        
        C --> D[Grant LF table/column permissions to all consumers]
        D --> E[Grant CREATE_TABLE + DESCRIBE on database if first table in DB]
        E --> F[Grant Data Location Permissions to writers/creators]
        F --> G[Register S3 location full registration]
        G --> H{Validate: Can all consumers access data?}
        
        H -->|Yes| I[Remove IAMAllowedPrincipals from table]
        H -->|No| R1[Rollback: Deregister location]
        R1 --> R2[Diagnose & fix LF grants]
        R2 --> G
        
        I --> J{All tables in this database migrated?}
        J -->|Yes| K[Remove IAMAllowedPrincipals from database]
        J -->|No| C
        K --> L{All databases migrated?}
        L -->|No| C
        L -->|Yes| M[Remove direct S3 access for migrated paths from IAM policies]
        M --> N[Clean up Glue Resource Policies]
        N --> O[Migration Complete]
    ```

## Steps

### Step 1 — Identify Consumers Per Table

Determine who is accessing each table and how frequently. See [How Do You Find Out Who Is Consuming What?](../lake-formation-transition-guide-single-account.md#how-do-you-find-out-who-is-consuming-what) in the overview.

### Step 2 — Prioritize Tables

- Start with tables that have the **fewest consumers** — these have the smallest blast radius.
- Adjust for business priority: you may choose to migrate high-priority or high-risk tables with low usage early.
- **Important:** Group tables by S3 location. When you register an S3 path, *all* tables under that path get credential vending simultaneously. Plan your batches around S3 prefixes to avoid accidentally migrating tables you're not ready for.

### Step 3 — For Each Table (or batch of tables sharing a location):

| Step | Action |
|------|--------|
| 3a | Grant LF **table/column permissions** to all identified consumers |
| 3b | Grant `CREATE_TABLE` + `DESCRIBE` on the **database** to principals who need it (if this is the first table in that database being migrated) |
| 3c | Grant **Data Location Permissions** to any users who write data or create tables at that S3 path |
| 3d | **Register the S3 location** (full registration). This activates credential vending for all tables under this path. |
| 3e | **Validate** — confirm all consumers can read/write as expected |
| 3f | **Remove IAMAllowedPrincipals** from the table |

### Step 4 — Database Cleanup

Once all tables in a database have been migrated, remove IAMAllowedPrincipals from the database itself.

### Step 5 — Final Cleanup

Once all databases are migrated:
- Remove direct S3 data access (for Glue table paths) from consumer IAM policies
- Remove any Glue Resource Policies that granted open catalog access

## Rollback — Plan #1

??? note "Click to expand rollback diagram"

    ```mermaid
    flowchart TD
        A[Consumer loses access after registration] --> B{Has IAMAllowedPrincipals been removed?}
        B -->|No| C[Deregister the S3 location]
        C --> D[Access reverts to IAM-based]
        D --> E[Diagnose: missing LF grant? missing Data Location Permission?]
        E --> F[Fix LF grants]
        F --> G[Re-register location]
        G --> H[Re-validate]
        
        B -->|Yes| I[Re-add IAMAllowedPrincipals to table/database]
        I --> J[Deregister the S3 location]
        J --> D
    ```

**Key principle:** Do NOT remove IAMAllowedPrincipals until you have validated access post-registration. Registration + validation is your safety net — keep IAMAllowedPrincipals in place until you're confident.
