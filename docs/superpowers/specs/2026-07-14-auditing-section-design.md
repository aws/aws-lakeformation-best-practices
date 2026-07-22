# Design: Auditing Section for Lake Formation Best Practices

**Date:** 2026-07-14
**Status:** Approved (pending spec review)

## Goal

Add a new top-level **Auditing** section to the AWS Lake Formation Best Practices
mkdocs site. The section teaches readers how to build end-to-end audit trails for
data-lake access by combining Lake Formation `GetDataAccess` CloudTrail events with
S3 access records (CloudTrail S3 Data Events or S3 Access Logs), including sample
CloudTrail events per engine and ready-to-run Athena queries that stitch the two
together.

## Context

- Project is an mkdocs (Material theme) site. `docs_dir: content`.
- Existing sections live in `content/<topic>/` folders, each with an `overview.md`
  plus topic pages, wired into `nav:` in `mkdocs.yml`.
- Pages are prose best-practices with admonitions, fenced code blocks (copy enabled),
  tables, and liberal links to public AWS documentation. Mermaid is available via
  `pymdownx.superfences`.
- Source material: an attached document (`Auditing in LF - CloudTrail Examples.pdf`)
  containing `GetDataAccess` CloudTrail examples for multiple engines in Full Table
  Access (FTA) and Fine-Grained Access Control (FGAC) modes, S3 CloudTrail and S3
  Access Log samples, and two full Athena stitching queries.

## Structure

New folder `content/auditing/` with four pages:

```
content/auditing/
  overview.md                    # Why audit; what LF + S3 auditing gives you; page map
  lake-formation-cloudtrail.md   # GetDataAccess events, one example per engine, field guide
  s3-audit-events.md             # CloudTrail Data Events vs S3 Access Logs — decision guide
  querying-audit-data.md         # Both Athena queries up top + limitations callout
```

Nav insertion in `mkdocs.yml`, placed after the `LF-Tags` block:

```yaml
- 'Auditing':
  - 'Overview': 'auditing/overview.md'
  - 'Lake Formation CloudTrail Events': 'auditing/lake-formation-cloudtrail.md'
  - 'Auditing S3 Access': 'auditing/s3-audit-events.md'
  - 'Querying Audit Data': 'auditing/querying-audit-data.md'
```

Add a bullet to `content/index.md`'s topic list linking to `auditing/overview.md`.

## Data Sanitization (applies to every example on every page)

All CloudTrail/S3 examples MUST be scrubbed. A short **scrubbing legend** appears near
the top of `lake-formation-cloudtrail.md` (and is referenced from other pages) stating
these are sanitized samples and listing the placeholder conventions:

| Real value type | Placeholder used |
|---|---|
| AWS account ID | `111122223333` |
| S3 bucket name | `amzn-s3-demo-bucket` (AWS-recommended example bucket name) |
| Access key ID (`accessKeyId`) | redacted / `AKIAIOSFODNN7EXAMPLE`-style example |
| Principal ID (`AROA...`, `:sessionId`) | shortened example value |
| Request/event IDs, `x-amz-id-2` | example UUIDs |
| Region | keep `us-east-2` (not sensitive) |
| IP addresses | example/service endpoint values |

Applies to: account IDs, bucket names, `accessKeyId`, `principalId`, session names,
request IDs, `x-amz-id-2`, and IP addresses. Engine/service identifiers
(`sourceIPAddress` like `athena.amazonaws.com`, `platformType`) are kept as-is because
they are documented, non-sensitive, and needed for auditing.

## Page Contents

### overview.md

- **Why auditing matters**: validating least-privilege grants, detecting policy drift
  and anomalous access, producing compliance evidence, incident forensics.
- **What Lake Formation auditing adds over IAM alone**: `GetDataAccess` captures the
  *authorization decision* plus query/engine context (which engine, which query,
  which principal, which resource, FGAC vs FTA) that raw S3 data events lack.
- **The end-to-end picture**: LF authorizes (`GetDataAccess`) → engine assumes a
  session → S3 objects are read. Full auditing means joining these.
- **Page map** linking the three sub-pages.

### lake-formation-cloudtrail.md

- Explanation of the `GetDataAccess` event: what triggers it, `eventSource:
  lakeformation.amazonaws.com`, that it is a management/read event.
- **Scrubbing legend** (table above).
- **One scrubbed example per engine**, in FTA and FGAC where the engine supports both:
  - EMR Serverless — FTA (table read) and FGAC (table read)
  - EMR on EC2 — FTA and FGAC
  - Athena (v3) — FGAC (note: FTA not supported)
  - Glue ETL — FGAC
  - Redshift Provisioned
  - Redshift Serverless
- **Field-reference table** grouped into three areas the reader needs for auditing:
  - **Identity**: `userIdentity.arn`, `userIdentity.sessionContext.sessionIssuer.arn`,
    `additionalEventData.lakeFormationPrincipal`,
    `additionalEventData.lakeFormationRoleSessionName` (call out that the session name
    is the join key to S3 events).
  - **Source/engine**: `sourceIPAddress`, `userAgent`, `requesterService`,
    `requestParameters.auditContext.additionalAuditContext.platformType`,
    `jobIdentifier`.
  - **Resource**: `requestParameters.tableArn` vs `requestParameters.dataLocations`,
    `permissions` (e.g. `SELECT`), `supportedPermissionTypes` /
    `CELL_FILTER_PERMISSION`, `cellLevelSecurityEnforced`. The **FTA vs FGAC vs
    direct-S3 signals** are explained here (FGAC → `cellLevelSecurityEnforced: true`
    and/or `CELL_FILTER_PERMISSION`; direct-S3/data-location access → `dataLocations`
    present instead of `tableArn`; `LakeFormationAuthorizedCaller` tag).
- **Quick-reference cheat table**: each engine → its `sourceIPAddress` / `userAgent` /
  `platformType` signature, so a reader can identify the producing engine from a log line.
- **Notes/caveats** from source doc: Table Direct S3 via LF is not supported when FGAC
  is enabled on EMR/Glue; EMR/Glue do not support FTA mode with LF for Iceberg tables.

### s3-audit-events.md

- Framing: to see which S3 objects were actually read you need S3-level auditing;
  two options exist.
- **Decision-guide comparison table**: CloudTrail S3 Data Events vs S3 Server Access
  Logs across: cost model, latency/delivery time, level of detail, completeness/
  guarantees, ability to join to LF via session name, setup complexity.
- **Recommendation**: state a clear "use X when…" recommendation.
- **Cost optimization for CloudTrail Data Events**: you can scope a data event
  selector to specific buckets/prefixes and to specific S3 operations
  (`GetObject`/`PutObject`/`DeleteObject`/`ListBucket`) to keep volume and cost low.
- Links to public docs for enabling both mechanisms.

### querying-audit-data.md

- **Both Athena queries at the top of the page**:
  1. CloudTrail S3 Data Events joined to LF `GetDataAccess`.
  2. S3 Server Access Logs joined to LF `GetDataAccess`.
- Brief explanation of the **join logic**: `s3_sts_session_name` (parsed from the S3
  event's assumed-role ARN) = `lakeFormationRoleSessionName` from the LF event; how
  `access_method` (LF vs DIRECT) and `lf_source` (engine) are derived.
- **Session-name linkage mermaid diagram**: `GetDataAccess` → assumed session
  (`AWSLF-…` session name) → S3 `GetObject`, showing where linkage exists and where it
  breaks for direct-S3 access.
- **Prominent limitations admonition** (`!!! warning`): direct S3 access via LF does
  not populate the LF session name, so those rows cannot be joined back to LF events;
  plus the FTA/FGAC caveats affecting what appears in the logs.
- **Operationalization tips**: run the join on a schedule (daily/hourly ETL into a
  curated table) vs on-demand; partition CloudTrail and Access-Log tables by date to
  control scan cost.

## Enhancements Included

1. Scrubbing legend (in `lake-formation-cloudtrail.md`, referenced elsewhere).
2. Operationalization tips (in `querying-audit-data.md`).
3. Session-name linkage mermaid diagram (in `querying-audit-data.md`).
4. Quick-reference engine cheat table (in `lake-formation-cloudtrail.md`).

## Out of Scope (YAGNI)

- No new images/screenshots beyond the one mermaid diagram.
- No reproduction of every raw JSON blob from the PDF — one representative example per
  engine/mode only.
- No changes to existing sections.
- No automation/scripts for log setup — link to public docs instead.

## Testing / Verification

- `mkdocs build` succeeds with no warnings about the new pages or nav entries.
- All internal links resolve; all four pages appear in nav in the intended order.
- Spot-check that no real account IDs, bucket names, access keys, or principal IDs
  from the source doc remain in any example.
