# Advanced Access Control Patterns

Lake Formation natively enforces access control at the database, table, and column level, through grants, LF-Tags, and row and cell filters. Most authorization requirements fit within that set. Some do not: masking that varies by consuming role, redaction tied to external attributes, or transformations the filter model cannot express.

This section collects patterns for that remainder. Each one composes native Lake Formation features with Glue ETL jobs and Glue Data Catalog views to reach the access-control behavior a requirement demands. 

The first pattern here is [Data Masking](data-masking/overview.md), which covers three ways to return masked values: precomputed by a Glue ETL job and split by Lake Formation grants, applied by a Data Catalog view at query time, or applied per caller by a view that joins an entitlement table.
