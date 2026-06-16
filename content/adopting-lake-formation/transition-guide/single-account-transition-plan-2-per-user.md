# Transition Plan #2 — Per User Transition

In this approach, you migrate **user by user** (or cohort by cohort). Hybrid access mode allows you to opt individual principals into Lake Formation credential vending while others continue using IAM. This gives you surgical control and per-user rollback.

## Flow Diagram

??? note "Click to expand flow diagram"

    ```mermaid
    flowchart TD
        A[Select a cohort of users] --> B[Look up tables they access via CloudTrail]
        B --> C[Identify if users create tables and where]
        C --> D[Grant Data Location Permissions for write paths]
        D --> E[Grant CREATE_TABLE on relevant databases]
        E --> F[Grant LF table/column permissions to cohort users]
        
        F --> G{Are table S3 locations already registered as Hybrid?}
        G -->|No| H[Register S3 locations as Hybrid]
        G -->|Yes| I[Skip — already registered]
        H --> I
        
        I --> J[Create LakeFormation Opt-In for each user + resource]
        J --> K{Validate: queries work? CREATE TABLE works?}
        
        K -->|Yes| L{More cohorts to migrate?}
        K -->|No| R1[Remove Opt-In for affected user]
        R1 --> R2[Diagnose & fix]
        R2 --> J
        
        L -->|Yes| A
        L -->|No| M[All users opted in for all locations]
        M --> N[Convert Hybrid locations to full registration]
        N --> O[Remove IAMAllowedPrincipals from tables and databases]
        O --> P[Remove direct S3 access from IAM policies]
        P --> Q[Migration Complete]
    ```

## Decision Tree — New Users During Migration

??? note "Click to expand decision tree"

    ```mermaid
    flowchart TD
        A[New user needs table access] --> B{Is the table's location in Hybrid mode?}
        B -->|No| C[User uses IAM as normal — not yet migrated]
        B -->|Yes| D{Is this user part of a migration cohort?}
        D -->|Yes| E[Grant LF permissions + Opt-In immediately]
        D -->|No| F[Recommended: Opt-in new users immediately to avoid growing backlog]
    ```

## Steps

### Step 1 — Select a Cohort

Choose a group of users to migrate. Good candidates for the first cohort:
- Users with limited table access (small blast radius)
- A single team that owns their own data
- Power users who can help validate quickly

### Step 2 — Identify Table Access and Write Patterns

- Look up all tables the cohort accesses (CloudTrail `GetTable`, `GetPartitions`, etc.)
- Check if these users **create tables** — look for `CreateTable`, `UpdateTable` calls
- Note the S3 paths where they create tables

### Step 3 — Grant Permissions

| Step | Action |
|------|--------|
| 3a | Grant **Data Location Permissions** for S3 paths where users write/create tables. If tables location is a prefix of the database's location, you can ignore this step as the permission is implicitly applied. |
| 3b | Grant `CREATE_TABLE` permission in LF on relevant databases |
| 3c | Grant LF **table/column permissions** to the cohort's users for all their tables |
| 3d | Ensure `lakeformation:GetDataAccess` is in their IAM policies (should already be done in prerequisites) |

### Step 4 — Register and Opt-In

- Register the tables' S3 locations as **Hybrid Access Mode** locations (if not already registered)
  - Hybrid Access Mode means: non-opted-in users continue on IAM, opted-in users use LF credential vending
- **Create LakeFormation Opt-In** (`CreateLakeFormationOptIn`) for each user + resource combination
  - This is the step that "flips" the user to LF where IAMAllowedPrincipal permission is ignore, and credential vending is enabled (if data lake location is registered)
- Keep IAMAllowedPrincipals **enabled** on databases — do not put databases in Hybrid mode (adds complexity with no value)

> **Note:** Location registration is a one-time step. If a location was registered as Hybrid during a previous cohort's migration, you only need to grant LF permissions and create opt-ins for the new cohort.

### Step 5 — Validate

- Run representative queries as the opted-in users
- Verify `CREATE TABLE` works for users who create tables
- Confirm that **non-opted-in users are unaffected** (still using IAM)
- If issues arise → remove the opt-in (see Rollback below)

### Step 6 — Repeat

Select the next cohort and repeat Steps 1–5.

### Step 7 — Completion

Once ALL users of a given location are opted in:
1. Convert from Hybrid to **full registration** (remove Hybrid flag)
2. Remove IAMAllowedPrincipals from tables and databases
3. Remove direct S3 access (for Glue table paths) from consumer IAM policies

**"Done" signal:** When `ListLakeFormationOptIns` shows all known consumers are opted in AND CloudTrail shows no non-opted-in access for a defined period (e.g., 7–14 days), you can safely convert to full registration.

## Rollback — Plan #2

??? note "Click to expand rollback diagram"

    ```mermaid
    flowchart TD
        A[User loses access after opt-in] --> B[Remove Opt-In: DeleteLakeFormationOptIn]
        B --> C[User immediately reverts to IAM-based access]
        C --> D[No impact to other users]
        D --> E[Diagnose: missing LF grant? Data Location Permission? CREATE_TABLE?]
        E --> F[Fix permissions]
        F --> G[Re-create Opt-In]
        G --> H[Re-validate]
    ```

**Key advantage:** Rollback is surgical. Removing an opt-in only affects that specific user — no other users or resources are impacted.
