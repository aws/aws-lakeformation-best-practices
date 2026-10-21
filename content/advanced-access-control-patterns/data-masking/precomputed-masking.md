# Precomputed Masking with Glue ETL

This page covers Approach 1 from the [data masking overview](overview.md): a Glue ETL job computes the masked values ahead of time and stores them, and Lake Formation grants decide which stored copy each persona reads.

Two designs share that description, and they differ only in where the masked values land:

- **Masked columns in the same table.** The job adds `name_masked`, `email_masked`, and `ssn_masked` alongside the raw columns. Column-level LF-Tag grants split the two sets between personas.
- **A separate masked table.** The job writes a second table carrying the same column names with the sensitive values replaced. Table-level grants point each persona at one table or the other.

The masking logic is identical in both, and so is the cost profile: pay once per ETL run, store the result, and readers pay nothing at query time. Pick between them on how consumers query the data. If a downstream tool is pinned to one table name, use the same-table variant. If you want one query definition to work for both personas, use the separate-table variant.

Both designs precompute, which is what separates this approach from the two view-based ones. If you would rather not run a pipeline or store a second copy of anything, read [Masking view](masking-view.md) first; it reaches the same outcome with no ETL.

Both variants change the name a consumer has to query: the same-table variant asks the restricted persona to select `ssn_masked` instead of `ssn`, and the separate-table variant asks it to select from `sales_masked.customers` instead of `sales.customers`. Neither is transparent to anything already reading the raw table. Every affected query, dashboard, saved search, BI dataset, and downstream job has to be found and edited before the restricted persona can use it.

## Variant A: masked columns in the same table

The job reads the seven-column `customers` table from the overview and writes back ten columns:

`customer_id`, `name`, `name_masked`, `email`, `email_masked`, `ssn`, `ssn_masked`, `city`, `country`, `loyalty_tier`

Every sensitive column gains a masked twin beside it. The public columns (`customer_id`, `city`, `country`, `loyalty_tier`) are untouched and unduplicated.

Pick one naming scheme for the twin and use it everywhere, without exception. This page uses a `_masked` suffix, but the convention matters less than its consistency: grant automation and consumer tooling derive the masked name from the raw name programmatically, so a single column that breaks the pattern, `ssn_redacted` instead of `ssn_masked`, forces every downstream script to special-case it.

### Glue ETL job

The job below applies the shared masking rules from the overview: SHA-256 for `name`, partial reveal for `email`, full redaction for `ssn`.

```python
from pyspark.sql.functions import col, concat, lit, sha2, split

masked = (
    customers
    .withColumn("name_masked", sha2(col("name"), 256))
    .withColumn(
        "email_masked",
        concat(
            split(col("email"), "@").getItem(0).substr(1, 1),
            lit("***@"),
            split(col("email"), "@").getItem(1),
        ),
    )
    .withColumn("ssn_masked", lit(None).cast("string"))
)
```

`ssn_masked` is written as `NULL`, which is full redaction. For the last-4 variant (`***-**-6789`), replace that line with:

```python
.withColumn("ssn_masked", concat(lit("***-**-"), col("ssn").substr(-4, 4)))
```

Write `masked` back to the `customers` table location on each run. The raw columns pass through unchanged; only the three `_masked` columns are computed.

### LF-Tag design

Every column in the table, raw or masked, carries a `Sensitivity` LF-Tag. This is not optional: an LF-Tag expression can only grant on tagged columns, so an untagged column is invisible to a tag-based grant and needs a separate column grant instead. Tagging all ten keeps the table under one grant mechanism.

| Column | `Sensitivity` value |
|---|---|
| `customer_id` | `Public` |
| `city` | `Public` |
| `country` | `Public` |
| `loyalty_tier` | `Public` |
| `name` | `PII` |
| `email` | `PII` |
| `ssn` | `PII` |
| `name_masked` | `Masked` |
| `email_masked` | `Masked` |
| `ssn_masked` | `Masked` |

Each persona then gets one LF-Tag expression:

- `MarketingAnalystRole`: `Sensitivity IN (Masked, Public)`, resolving to `customer_id`, `city`, `country`, `loyalty_tier`, `name_masked`, `email_masked`, `ssn_masked`.
- `DataEngineerRole`: `Sensitivity IN (PII, Public)`, resolving to `customer_id`, `city`, `country`, `loyalty_tier`, `name`, `email`, `ssn`.

Neither expression names a column directly. Adding an eleventh sensitive column later means tagging it `PII` and its twin `Masked`; both grants pick it up untouched.

This call grants `MarketingAnalystRole` its side of the split:

```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::111122223333:role/MarketingAnalystRole \
  --permissions "SELECT" \
  --resource '{
    "LFTagPolicy": {
      "CatalogId": "111122223333",
      "ResourceType": "TABLE",
      "Expression": [
        {
          "TagKey": "Sensitivity",
          "TagValues": ["Masked", "Public"]
        }
      ]
    }
  }'
```

The `DataEngineerRole` grant is the same shape with `TagValues` set to `["PII", "Public"]`.

!!! warning

    Lake Formation computes a principal's effective permissions as the union of every grant that applies to it, LF-Tag expressions and named-resource grants alike. If `MarketingAnalystRole` picks up a second grant carrying `Sensitivity=PII`, whether from a direct column grant, a database-level grant, or a second LF-Tag expression added by someone else later, it sees the raw `ssn`, `email`, and `name` columns in addition to the masked ones. The masking is not weakened in that case, it is void: nothing prevents the underlying value from being read.

    Each persona is correctly configured only when it ends up with exactly one variant of each column. Do not treat issuing the grant above as the end of the job. Check the principal's effective permissions after the grant lands, and re-check whenever another grant touches this table, since the union rule means a grant issued for an unrelated reason can reopen access this design was built to close.

### What consumers see

The two personas read different column sets from the same table, so a query written for one does not run for the other. A dashboard selecting `ssn_masked` fails for `DataEngineerRole`, which has no grant on that column; a dashboard selecting `ssn` fails for the analyst the same way. Neither persona sees all ten columns from `SELECT *`: each gets seven, the four public columns plus its own variant of the three sensitive ones. Two roles running the identical query against the identical table get result sets of the same width and different content, which catches people off guard the first time they compare notes.

## Variant B: a separate masked table

`sales.customers` keeps its seven columns untouched and is granted to `DataEngineerRole` only. A second table carries the same seven column names with `name`, `email`, and `ssn` masked, granted to `MarketingAnalystRole` only. Column names do not change between the two tables; only the values in three of them differ.

Where the second table lives is the one open decision:

- A separate database, `sales_masked`, makes the grant boundary obvious: a grant on the `sales` database or the `sales.customers` table cannot leak into `sales_masked`, and `MarketingAnalystRole` never sees `sales.customers` when it lists tables. The cost is another database to create, tag, and track.
- Both tables in one database, `sales.customers` and `sales.customers_masked`, is fewer objects to manage. The cost is that anyone who can list tables in `sales` sees the raw and masked names side by side, which invites someone to query the wrong one.

The rest of this section uses `sales_masked.customers`.

### Glue ETL job

The masked values overwrite the columns in place, so this job is shorter than Variant A's: no suffix, no extra columns, the same seven columns written to a different table.

```python
from pyspark.sql.functions import col, concat, lit, sha2, split

masked = (
    customers
    .withColumn("name", sha2(col("name"), 256))
    .withColumn(
        "email",
        concat(
            split(col("email"), "@").getItem(0).substr(1, 1),
            lit("***@"),
            split(col("email"), "@").getItem(1),
        ),
    )
    .withColumn("ssn", lit(None).cast("string"))
)
masked.write.mode("overwrite").saveAsTable("sales_masked.customers")
```

`customers` is the raw `sales.customers` DataFrame read at the start of the job. Nothing in the transform depends on which database the output lands in; the choice above only affects the `saveAsTable` argument.

### Grants

Both grants are table-level `SELECT` with no column lists. `DataEngineerRole` gets `sales.customers`; `MarketingAnalystRole` gets `sales_masked.customers`. There is no column set to keep in sync with a tag scheme, which makes this the easiest of the designs on this page to audit: read the two grants and you know who sees what.

```bash
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::111122223333:role/MarketingAnalystRole \
  --permissions "SELECT" \
  --resource '{"Table": {"CatalogId": "111122223333", "DatabaseName": "sales_masked", "Name": "customers"}}'
```

An LF-Tag variant scales better past the first table pair. Tag each table itself, not its columns, with `Sensitivity=PII` or `Sensitivity=Masked`, and grant the tag expression instead of naming each table:

```json
{"LFTagPolicy": {"CatalogId": "111122223333", "ResourceType": "TABLE", "Expression": [{"TagKey": "Sensitivity", "TagValues": ["Masked"]}]}}
```

A new raw/masked pair then needs one tag assignment on each table rather than two new named-resource grants.

### One query definition serves both personas

Because the column names are identical, a query written against `sales.customers` runs against `sales_masked.customers` unchanged. A BI tool's saved query or a parameterized ETL step points at one table or the other by swapping the database name; the `SELECT` list, joins, and column references stay as they are.

That is the advantage over Variant A, where the two personas see different column sets and a single query cannot serve both.

!!! warning

    The masked table only protects data that cannot also be read from the raw one. If `MarketingAnalystRole` ends up with read access to `sales.customers`, whether through a leftover grant nobody revoked, a database-level grant on `sales` that predates the split, or `IAMAllowedPrincipals` left enabled on the table, the masked copy accomplishes nothing: the analyst queries the raw table instead. Check the effective permissions on `sales.customers` whenever this design is in place, not only on `sales_masked.customers`.

## Cost and performance

Both variants compute the masking once per ETL run and store the result, so readers pay nothing for it. A masked column is a materialized column like any other, and a query selecting it scans exactly as a query against the raw column would.

The recurring cost is Glue job time plus storage, and it scales with the dataset and the pipeline schedule rather than with query volume. The two variants differ in how much of each:

- Variant A adds one column per masked column, and the job computes only those columns.
- Variant B duplicates every row and every public column alongside the masked ones, so it stores more and its job runtime scales with the full row count rather than with the three columns being masked.

That is the opposite profile from the two view-based approaches, which store nothing and pay at read time instead. A table queried constantly favours precomputing; a large table queried rarely does not.

## Trade-offs

Precomputed masking works on any engine that honors Lake Formation grants, because the mechanism is ordinary column-level or table-level grants rather than anything engine-specific. Nothing about it depends on a view, a SQL dialect, or a caller-identity function.

The costs are a pipeline and a stored second copy, in columns or in a whole table. Freshness is bounded by the last successful run, and in Variant B a lagging job means the analyst reads stale data while the engineer reads fresh data, silently, since nothing about a successful `SELECT` indicates how far behind a table is. Both variants also multiply with sensitivity levels: N levels means N column sets or N tables, each with its own grant and its own freshness check.

Neither variant masks per user. A persona sees the raw set or the masked set, decided by a grant. If entitlements need to vary per principal without a grant change, use [Masking view with an entitlement table](glue-views-entitlement-table.md) instead.
