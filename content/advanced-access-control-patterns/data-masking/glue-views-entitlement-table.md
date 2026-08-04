# Masking View with an Entitlement Table

This page covers Approach 3 from the [data masking overview](overview.md). It extends the [masking view](masking-view.md) with one change: instead of masking every reader identically, the view joins an entitlement table keyed on the caller's identity and decides per principal which columns to reveal.

Reach for it when a single view has to serve principals with different rights, or when entitlements change often enough that editing a table beats issuing a grant. If every restricted reader should see the same masked output, the simpler fixed-masking view is the better choice.

These are Data Catalog views, also called multi-dialect views, and they are not materialized views: nothing is precomputed or stored. A Data Catalog view is a stored query definition the engine resolves and re-executes on every read, which is what lets the masking depend on who is running the query at that moment.

!!! note

    This pattern reads the caller's identity with `invoker_principal()`, which exists only in Athena engine version 3. Data Catalog views themselves can be read from Redshift and Spark as well, but those engines have no equivalent caller-identity function, so a query against this view from anywhere other than Athena cannot resolve who is asking. Restrict the view to Athena consumers. If a consumer must read the data through Redshift, EMR, or Glue, give it a [masking view](masking-view.md) with fixed masking, or [precomputed masking](precomputed-masking.md).

## The three roles

Three principals participate, and Lake Formation permissions land on each differently:

- **Lake Formation Admin.** Configures the Lake Formation permissions that make the rest of this work: the Definer's grantable `SELECT` on the base tables, and the Invokers' `SELECT` on the view.
- **Definer.** The IAM role that creates the view. It must hold full, grantable `SELECT` on every table the view definition references, because Lake Formation checks the Definer's permissions, not the Invoker's, when the view runs. The Definer must be an IAM role, not a user or group, and its trust policy must allow `sts:AssumeRole` for the AWS Glue and Lake Formation service principals.
- **Invoker.** Any principal the Lake Formation Admin grants `SELECT` on the view itself, after the view exists. `DataEngineerRole` and `MarketingAnalystRole` are both Invokers of `sales.customers_masked` in this example. Neither needs any permission on `sales.customers`: the Invoker queries the view and never touches the base table's grants.

That last point is the security property the whole pattern rests on: querying `customers_masked` requires no permission on `customers`.

!!! warning

    That property only holds if it is the only path to the data. If an Invoker also holds `SELECT` on `sales.customers` directly, the masking is decorative: the Invoker can bypass the view and query the base table for raw values whenever it wants. Revoke base-table access from every persona this pattern is meant to mask, `MarketingAnalystRole` included, and check that no LF-Tag expression or database-level grant reopens it later.

## `invoker_principal()`

`invoker_principal()` returns a `VARCHAR` containing the ARN of the principal that ran the query calling it. That principal is an IAM role or an Identity Center identity. The role running the query must allow the `lakeformation:GetDataLakePrincipal` action, or the call fails.

The returned ARN is the role ARN, not the STS assumed-role ARN:

```text
Returned by invoker_principal():  arn:aws:iam::111122223333:role/MarketingAnalystRole
NOT returned (assumed-role form): arn:aws:sts::111122223333:assumed-role/MarketingAnalystRole/session-name
```

That distinction is what makes the entitlement table workable. The role ARN is stable across every session a principal opens, so the entitlement table keys on one row per principal, not one row per session. Key it on the assumed-role form instead and every new session mints a session name the table has never seen, and every join misses.

## Entitlement table

`sales.pii_entitlements` holds one row per principal:

| `principal_arn` | `role_name` | `can_see_pii` | `can_see_name` | `can_see_email` | `can_see_ssn` |
|---|---|---|---|---|---|
| `arn:aws:iam::111122223333:role/DataEngineerRole` | `DataEngineerRole` | `true` | `true` | `true` | `true` |
| `arn:aws:iam::111122223333:role/MarketingAnalystRole` | `MarketingAnalystRole` | `false` | `false` | `false` | `false` |

Two granularities are available at once. `can_see_pii` is a single flag for an all-or-nothing split: entitled or not, with no distinction between `name`, `email`, and `ssn`. The three `can_see_*` columns split that further, so one principal could see `name` but not `ssn`. The view definition below joins on the per-column flags. `role_name` exists for whoever reads the table; the join key is `principal_arn`, and `role_name` never appears in the view's `WHERE` or `JOIN` clauses.

## View definition

```sql
CREATE OR REPLACE PROTECTED MULTI DIALECT VIEW sales.customers_masked
SECURITY DEFINER
AS
SELECT
    c.customer_id,
    CASE WHEN COALESCE(e.can_see_name, false)
         THEN c.name
         ELSE to_hex(sha256(to_utf8(c.name))) END     AS name,
    CASE WHEN COALESCE(e.can_see_email, false)
         THEN c.email
         ELSE concat(substr(split_part(c.email, '@', 1), 1, 1),
                     '***@', split_part(c.email, '@', 2)) END AS email,
    CASE WHEN COALESCE(e.can_see_ssn, false)
         THEN c.ssn ELSE NULL END                     AS ssn,
    c.city,
    c.country,
    c.loyalty_tier
FROM sales.customers c
LEFT JOIN sales.pii_entitlements e
       ON e.principal_arn = invoker_principal()
```

`SECURITY DEFINER` is required syntax for a Data Catalog view. `UNPROTECTED` Data Catalog views are not supported either, so `PROTECTED` is not optional.

The `LEFT JOIN` is deliberate. A principal with no row in `pii_entitlements` produces `NULL` for every `can_see_*` column, `COALESCE` turns each `NULL` into `false`, and every `CASE` branch falls to the masked output. A new hire who has not yet been added to the entitlement table sees masked values by default, not raw ones. The pattern fails closed.

This is Athena SQL, specifically the Trino dialect, so the function names differ from the PySpark jobs on the [precomputed masking](precomputed-masking.md) page even though the masking rules are identical. Two differences are worth calling out, because both are easy to get wrong when porting an expression between the two:

| Rule | PySpark | Athena SQL (Trino) |
|---|---|---|
| Hash `name` | `sha2(col("name"), 256)` | `to_hex(sha256(to_utf8(c.name)))` |
| Split `email` | `split(col("email"), "@").getItem(0)` | `split_part(c.email, '@', 1)` |

Trino has no `sha2` function. Its `sha256` takes `varbinary` and returns `varbinary`, so the column has to be encoded with `to_utf8` on the way in and rendered with `to_hex` on the way out; skipping either one is a type error at view creation. Trino's `split_part` is also 1-indexed, where PySpark's `getItem` starts at 0.

Before running the `CREATE OR REPLACE` above against a real environment, append `SHOW VIEW JSON` to the same statement to dry-run it: Athena validates the definition and, if it is valid, returns the Glue table JSON that would represent the view, without creating anything.

## Operational notes

`sales.pii_entitlements` is an access-control policy stored as data, not a data table. Restrict it in Lake Formation the same way you would restrict a grant, and do not grant Invoker principals `SELECT` on it: they need no access to it, since the Definer's permissions are what let the view's join run.

Adding a person is an `INSERT` into that table, not a Lake Formation grant. That is the pattern's main convenience, and its main audit gap. Lake Formation's [audit trail](../../auditing/lake-formation-cloudtrail.md) records `GetDataAccess` events for the view being read; it does not record which branch of the `CASE` expression a given query hit, so those events will not tell you who saw a raw `ssn` on a given day. If that evidence matters, version or log changes to `pii_entitlements` separately.

The documented constraints on Data Catalog views bind this pattern specifically:

- A view definition can reference up to 10 tables, and `pii_entitlements` counts as one of them alongside `customers` and anything else joined in.
- The view cannot reference other views, database resource links, or table resource links.
- Base tables must not carry the `IAMAllowedPrincipals` permission; if one does, the view fails with `Multi Dialect views may only reference tables without IAMAllowedPrincipals permissions`.
- Every base table's S3 location must be registered as a Lake Formation data lake location, or the view fails with `Multi Dialect views may only reference Lake Formation managed tables`.
- UDFs are not supported in the view definition, which is why the masking above uses built-in functions (`to_hex`, `sha256`, `to_utf8`, `concat`, `substr`, `split_part`) rather than a custom function.
- Tables in Amazon S3 table buckets (S3 Tables) are not supported as base tables, so neither `customers` nor `pii_entitlements` can live in one.

For the current list, see [Data Catalog views considerations and limitations](https://docs.aws.amazon.com/lake-formation/latest/dg/views-notes.html) in the Lake Formation Developer Guide, plus the [Athena-specific limitations](https://docs.aws.amazon.com/athena/latest/ug/views-glue.html#views-glue-limitations).

## Trade-offs

No duplicated data: one table, one view, no second copy of `customers` to keep in sync. No ETL job to schedule or monitor. Both personas see the same column names, so one query definition works for both. Entitlement changes take effect on the next query, since there is nothing to re-run or re-deploy.

The costs run the other way. Readers must be on Athena, because `invoker_principal()` exists nowhere else, which makes this the only approach here with a hard engine restriction. Part of the authorization model also moves out of Lake Formation grants and into `pii_entitlements` as data, so answering "who can see the real `ssn`" means reading a table in addition to reading Lake Formation grants.

This is also the most expensive of the three at read time. [Precomputed masking](precomputed-masking.md) pays once per ETL run and stores the result, so its readers pay nothing; a [fixed masking view](masking-view.md) evaluates masking expressions per query; this pattern does that and adds a join to `pii_entitlements` plus an `invoker_principal()` call on top. The extra work repeats on every query rather than per ETL run, adding scanned bytes on engines billed that way and compute time on the others. A view over a large table queried a few times a day is cheap by comparison; the same view behind dashboards refreshing all day may not be. Keeping `pii_entitlements` small enough for the engine to broadcast it limits the join cost, though the per-row `CASE` evaluation remains either way.

Take the per-principal masking only if you need it. If every restricted reader can see the same masked values, a [fixed masking view](masking-view.md) delivers that outcome without the join, the entitlement table, or the Athena restriction.
