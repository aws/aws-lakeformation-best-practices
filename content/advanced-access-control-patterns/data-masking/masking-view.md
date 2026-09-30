# Masking View

This page covers Approach 2 from the [data masking overview](overview.md): a Glue Data Catalog view selects from `sales.customers` and applies the masking in its `SELECT` list. Lake Formation grants the view to the restricted persona and the base table to the privileged one. No pipeline runs, and nothing is stored.

It produces the same masked output as [precomputed masking](precomputed-masking.md) without the ETL job, without a second copy of the data, and without a staleness window, because the masking is applied to current data every time the view is read.

The column names inside the view match the table, so the body of an existing query keeps working, but the object it selects **from** changes: the restricted persona has to query `sales.customers_masked` rather than `sales.customers`. Anything already reading the raw table needs that edit, so inventory the dashboards, saved queries, BI datasets, and jobs involved before switching a persona over.

## Two objects, two grants

The design has one table and one view:

- `sales.customers`, the raw table, granted to `DataEngineerRole`.
- `sales.customers_masked`, a Data Catalog view over it, granted to `MarketingAnalystRole`.

The view carries the same seven column names as the table, with `name`, `email`, and `ssn` masked in its definition. Nothing about `sales.customers` changes, so existing consumers of the raw table are unaffected by adding the view.

`MarketingAnalystRole` needs no permission on `sales.customers`. That is the property the whole approach depends on, and it comes from definer semantics: the view runs with the permissions of the role that created it, not the role querying it.

## Definer semantics

Three principals participate, and Lake Formation permissions land on each differently:

- **Lake Formation Admin.** Configures the permissions the rest of this needs: the Definer's grantable `SELECT` on `sales.customers`, and the Invoker's `SELECT` on the view.
- **Definer.** The IAM role that creates the view. It must hold full, grantable `SELECT` on every table the definition references, because the engine uses the Definer's permissions to read those tables when the view runs. The Definer must be an IAM role, and its trust policy must allow `sts:AssumeRole` for the AWS Glue and Lake Formation service principals.
- **Invoker.** Any principal granted `SELECT` on the view. `MarketingAnalystRole` is the Invoker here, and it queries `customers_masked` without any grant on `customers`.

!!! warning

    The masking holds only if the view is the only path to the data. If `MarketingAnalystRole` also holds `SELECT` on `sales.customers`, whether from a leftover grant, a database-level grant on `sales`, or `IAMAllowedPrincipals` left enabled, it can bypass the view and read raw values directly. Revoke base-table access from every persona this view is meant to mask, and re-check effective permissions whenever another grant touches `sales.customers`.

## View definition

```sql
CREATE OR REPLACE PROTECTED MULTI DIALECT VIEW sales.customers_masked
SECURITY DEFINER
AS
SELECT
    customer_id,
    to_hex(sha256(to_utf8(name)))                  AS name,
    concat(substr(split_part(email, '@', 1), 1, 1),
           '***@', split_part(email, '@', 2))       AS email,
    CAST(NULL AS VARCHAR)                           AS ssn,
    city,
    country,
    loyalty_tier
FROM sales.customers
```

Every keyword in the first two lines is required. `PROTECTED` is mandatory because Data Catalog views cannot be created any other way, and `SECURITY DEFINER` is what puts definer semantics in force. `OR REPLACE` updates an existing view, though it fails if dialects from other engines are present in the view; more on that below.

`ssn` is cast to `VARCHAR` rather than written as a bare `NULL` so the view's column type is declared rather than inferred. For the last-4 variant instead of full redaction, replace that line with:

```sql
concat('***-**-', substr(ssn, -4))                  AS ssn,
```

Append `SHOW VIEW JSON` to the statement to dry-run it. Athena validates the definition and returns the Glue table JSON it would create, without creating anything.

The masking expressions here are Trino syntax, because the view is being created from Athena. Trino has no `sha2` function; its `sha256` takes and returns `varbinary`, so the value is encoded with `to_utf8` going in and rendered with `to_hex` coming out. Skipping either is a type error at creation time. Compare the PySpark form on the [precomputed masking](precomputed-masking.md) page, which expresses the identical rule as `sha2(col("name"), 256)`.

## Granting the view

Grant `SELECT` on the view exactly as you would on a table. The Invoker gets nothing on `sales.customers`.

```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::111122223333:role/MarketingAnalystRole \
  --permissions "SELECT" \
  --resource '{"Table": {"CatalogId": "111122223333", "DatabaseName": "sales", "Name": "customers_masked"}}'
```

LF-Tags work on views too, so tagging the view `Sensitivity=Masked` and the base table `Sensitivity=PII` lets one tag expression per persona cover both objects, which scales better once several masked views exist.

## Reading the view from more than one engine

Data Catalog views are multi-dialect, and this is where the "MULTI DIALECT" keyword earns its place. Amazon Redshift, Athena engine version 3, Apache Spark on EMR Serverless, and Apache Spark on AWS Glue 5.0 all support Data Catalog views, so a view created from Athena can be read from the others.

The catch is that each engine reads its own SQL dialect. Creating the view from Athena stores a Trino-dialect definition, and `to_hex(sha256(to_utf8(...)))` is Trino syntax. To serve a Redshift consumer, add a Redshift-dialect definition to the same view with `ALTER VIEW`, expressing the same masking in Redshift SQL. Every dialect must reference the same tables, columns, and data types, so the dialects differ in function names while the view's schema stays fixed.

Plan for that before choosing this approach for a mixed-engine estate: one masking rule now has one definition per engine to keep in agreement. A single-engine estate never encounters the problem, and [precomputed masking](precomputed-masking.md) avoids it entirely by storing values that any engine can read without a dialect.

## Limitations

The documented constraints on Data Catalog views apply:

- A view definition can reference up to 10 tables.
- Views cannot reference other views, database resource links, or table resource links.
- Base tables must not carry the `IAMAllowedPrincipals` permission, or the view fails with `Multi Dialect views may only reference tables without IAMAllowedPrincipals permissions`.
- Every base table's S3 location must be registered as a Lake Formation data lake location, or the view fails with `Multi Dialect views may only reference Lake Formation managed tables`.
- UDFs are not supported in the definition, so the masking must be expressible in the engine's built-in functions.
- Tables in Amazon S3 table buckets (S3 Tables) are not supported as base tables. A table in a table bucket has to be masked another way, such as [precomputed masking](precomputed-masking.md).
- Athena reports an error for stale views, which happens when the view references tables or databases that no longer exist, when a referenced table's schema or metadata changes, or when a referenced table is dropped and recreated with a different schema. A column added to `sales.customers` does not appear in the view until the definition is updated.

For the current list, see [Data Catalog views considerations and limitations](https://docs.aws.amazon.com/lake-formation/latest/dg/views-notes.html) in the Lake Formation Developer Guide, plus the engine-specific limitations for [Athena](https://docs.aws.amazon.com/athena/latest/ug/views-glue.html#views-glue-limitations) and [Redshift](https://docs.aws.amazon.com/redshift/latest/dg/data-catalog-views-overview.html#data-catalog-views-considerations).

## Cost and performance

Nothing is stored and no pipeline runs, so this approach has no write-side cost. Each query against the view scans `sales.customers` and evaluates the masking expressions on the rows it returns, which is compute the reader pays for on every query.

That cost is smaller than the [entitlement table approach](glue-views-entitlement-table.md), which adds a join to the same work, and larger than [precomputed masking](precomputed-masking.md), where the values are already materialized and the reader pays nothing. For most workloads the difference is minor next to the base table scan; for a table under constant dashboard load, measure it before assuming so.

## Trade-offs

No ETL, no duplicated data, no staleness window, and readers see current values with the same column names the raw table uses, so one query definition works against either object. Adding a masked view to an existing table changes nothing about that table.

The masking is fixed in the view definition, which is the main limitation. Every Invoker of `customers_masked` sees the same masked output; there is no per-principal variation. Two personas needing different masking need two views, and N sensitivity levels need N views. If entitlements must vary per principal, or change without a redefinition, use [Masking view with an entitlement table](glue-views-entitlement-table.md), which keys the masking off the caller's identity at query time.

The other costs are the dialect work in a mixed-engine estate, and that the view must remain the only path to the data for the masking to mean anything.
