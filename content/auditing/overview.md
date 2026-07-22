# Auditing in Lake Formation

This section covers how to audit data access in a Lake Formation-governed data lake — capturing who accessed what, when, through which engine, and whether the access was authorized — so you can meet compliance requirements and investigate incidents with confidence. We have provided sample Amazon Athena queries in [Querying Audit Data](querying-audit-data.md), which correlates Lake Formation authorization events with the underlying S3 object access to produce an end-to-end engine-to-S3 mapping — for each S3 object read or written, it shows which principal was authorized, by which engine, against which catalog resource.

## Why auditing matters

Auditing is how you verify that your access controls are working as intended. Specifically, it lets you:

- **Validate least-privilege grants.** Confirm that principals are exercising only the permissions they have been granted, and that unused grants can be safely removed.
- **Detect policy drift and anomalous access.** Identify unexpected principals, engines, or access patterns before they become security incidents.
- **Produce compliance evidence.** Demonstrate to auditors that access to sensitive data is controlled and recorded.
- **Support incident forensics.** Reconstruct the exact sequence of access events when an incident needs investigation.

## What Lake Formation auditing adds over IAM alone

When a query engine such as Athena, Redshift Spectrum, or EMR requests access to a Lake Formation-governed resource, Lake Formation evaluates the request and emits a `GetDataAccess` CloudTrail event. This event captures the *authorization decision* together with rich context that raw S3 data events alone cannot provide:

- **Which principal** made the request (IAM role or user ARN)
- **Which resource** was authorized (database, table, or column set)
- **Which engine** submitted the request and, where available, the associated query ID
- **Whether Fine-Grained Access Control (FGAC) or Full Table Access (FTA) applied** — distinguishing between column- or row-level filtering and pass-through access

This context is essential for understanding *why* a set of S3 objects was read, not just that it was.

## The end-to-end picture

A governed data access flows through three stages:

1. A query engine calls Lake Formation's `GetDataAccess` API. Lake Formation evaluates the grants, records the authorization decision in CloudTrail, and — if approved — vends a short-lived, scoped session credential.
2. The engine uses that credential to read or write the underlying S3 objects. S3 records this access in its own audit trail (CloudTrail S3 Data Events or S3 Server Access Logs).
3. Full auditing requires **correlating** the Lake Formation authorization events with the S3 access records. Lake Formation stamps the credential it vends with a distinctive session name (beginning `AWSLF-`); that same session name appears in the `GetDataAccess` event (as `lakeFormationRoleSessionName`) and in the identity of the S3 requests made with the credential, which is what makes the join possible.

## In this section

- [Lake Formation CloudTrail Events](lake-formation-cloudtrail.md) — the `GetDataAccess` event structure, example events per query engine, and a field reference.
- [S3 Audit Events](s3-audit-events.md) — CloudTrail S3 Data Events vs. S3 Server Access Logs, and how to choose between them.
- [Querying Audit Data](querying-audit-data.md) — Athena queries that join S3 access records to Lake Formation authorization events.
