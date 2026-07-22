# Auditing Section Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new top-level "Auditing" section (four mkdocs pages) to the Lake Formation Best Practices site, teaching readers to build end-to-end audit trails by joining Lake Formation `GetDataAccess` CloudTrail events with S3 access records.

**Architecture:** Four prose markdown pages under `content/auditing/`, following the existing folder+overview convention. Wired into `mkdocs.yml` nav after LF-Tags and linked from `content/index.md`. All CloudTrail/S3 examples are scrubbed with placeholder values. One mermaid diagram on the querying page.

**Tech Stack:** mkdocs 1.6.0 + mkdocs-material, run from `.venv`. Verification is `mkdocs build --strict` (catches broken links/nav) plus manual scrub/content checks.

**Spec:** `docs/superpowers/specs/2026-07-14-auditing-section-design.md`

---

## Conventions For Every Task

- **Activate venv first** in each build command: `source .venv/bin/activate`.
- **Verification command** (the "test"): `mkdocs build --strict` must exit 0. `--strict` turns broken internal links and nav warnings into errors, so it is our failing/passing signal.
- **Scrubbing rules** (apply to ALL examples):
  - AWS account ID → `111122223333`
  - S3 bucket → `amzn-s3-demo-bucket`
  - `accessKeyId` → `AKIAIOSFODNN7EXAMPLE`
  - Principal IDs (`AROA…`), STS session suffixes → short example values (e.g. `AROAEXAMPLEID`, `:EXAMPLE-SESSION`)
  - Request IDs / event IDs / `x-amz-id-2` → example UUIDs / `EXAMPLE...`
  - IP addresses → keep AWS service endpoints as-is (`athena.amazonaws.com` etc.); replace real customer IPs (e.g. `18.218.40.42`) with `203.0.113.10`
  - Region `us-east-2` kept as-is
  - Keep engine identifiers as-is: `sourceIPAddress` service endpoints, `platformType`, `requesterService`
- **Editorial decisions already made:**
  - s3-audit-events recommendation: recommend **CloudTrail S3 Data Events (bucket/prefix + operation-filtered)** as primary for auditability (clean session-name join, near-real-time); **S3 Server Access Logs** as lower-cost/high-volume alternative.
  - Athena table names: keep from source queries but scrub account ID to `111122223333`; add a note telling readers to substitute their own table names.
- **Public doc links to use** (verify they resolve; these are the canonical pages):
  - LF CloudTrail: `https://docs.aws.amazon.com/lake-formation/latest/dg/logging-using-cloudtrail.html`
  - CloudTrail S3 data events: `https://docs.aws.amazon.com/AmazonS3/latest/userguide/enable-cloudtrail-logging-for-s3.html`
  - CloudTrail advanced event selectors (filtering): `https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-data-events-with-advanced-event-selectors.html`
  - S3 Server Access Logging: `https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html`
  - Athena CloudTrail table setup: `https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html`
  - Athena S3 access logs table: `https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-s3-access-logs-to-identify-requests.html`

---

## Task 1: Scaffold folder, nav, and index link

Get the four empty-but-valid pages wired into nav and building strict-clean before writing prose. This locks in structure and gives every later task a passing baseline.

**Files:**
- Create: `content/auditing/overview.md`
- Create: `content/auditing/lake-formation-cloudtrail.md`
- Create: `content/auditing/s3-audit-events.md`
- Create: `content/auditing/querying-audit-data.md`
- Modify: `mkdocs.yml` (add Auditing nav block after the `LF-Tags` block)
- Modify: `content/index.md` (add topic bullet)

- [ ] **Step 1: Create the four pages with a minimal H1 + one-line intro each**

Each file gets a valid H1 so nav resolves. Example for `overview.md`:

```markdown
# Auditing in Lake Formation

Overview of auditing data access in Lake Formation. *(content added in later tasks)*
```

Give the other three placeholder H1s: `# Lake Formation CloudTrail Events`, `# Auditing S3 Access`, `# Querying Audit Data`.

- [ ] **Step 2: Add the nav block to `mkdocs.yml`**

Insert directly after the `- 'LF-Tags':` block's last child (`limitations.md`) and before `markdown_extensions:`. Match the existing nav indentation exactly (top-level entries at 2 spaces under `nav:`, children at 4 spaces) — check the surrounding lines in `mkdocs.yml` before inserting:

```yaml
  - 'Auditing':
    - 'Overview': 'auditing/overview.md'
    - 'Lake Formation CloudTrail Events': 'auditing/lake-formation-cloudtrail.md'
    - 'Auditing S3 Access': 'auditing/s3-audit-events.md'
    - 'Querying Audit Data': 'auditing/querying-audit-data.md'
```

- [ ] **Step 3: Add the topic bullet to `content/index.md`**

Append to the bulleted topic list:

```markdown
- [Best Practices for auditing data access in Lake Formation](auditing/overview.md)
```

- [ ] **Step 4: Verify strict build passes**

Run: `source .venv/bin/activate && mkdocs build --strict`
Expected: exit 0, no warnings. Confirms all four pages resolve in nav and the index link is valid.

- [ ] **Step 5: Commit**

```bash
git add mkdocs.yml content/index.md content/auditing/
git commit -m "Scaffold Auditing section: pages, nav, index link"
```

---

## Task 2: overview.md

**Files:**
- Modify: `content/auditing/overview.md`

- [ ] **Step 1: Write the overview content**

Sections (prose, match the tone of `data-sharing/overview.md`):
- Intro paragraph: what this section covers.
- **Why auditing matters**: validating least-privilege grants, detecting policy drift / anomalous access, compliance evidence, incident forensics.
- **What Lake Formation auditing adds over IAM alone**: `GetDataAccess` captures the authorization decision + query/engine context (engine, query, principal, resource, FGAC vs FTA) that raw S3 events lack.
- **The end-to-end picture**: LF authorizes (`GetDataAccess`) → engine assumes a session → S3 objects read; full auditing = joining these.
- **In this section** — bulleted links to the three sub-pages:
  - `[Lake Formation CloudTrail Events](lake-formation-cloudtrail.md)`
  - `[Auditing S3 Access](s3-audit-events.md)`
  - `[Querying Audit Data](querying-audit-data.md)`

- [ ] **Step 2: Verify strict build passes**

Run: `source .venv/bin/activate && mkdocs build --strict`
Expected: exit 0 (relative links to sibling pages resolve).

- [ ] **Step 3: Commit**

```bash
git add content/auditing/overview.md
git commit -m "Add Auditing overview page"
```

---

## Task 3: lake-formation-cloudtrail.md

The largest page. Build it in two commits: examples first, then field guide + cheat table + notes.

**Files:**
- Modify: `content/auditing/lake-formation-cloudtrail.md`

**Source examples** (from the provided CloudTrail doc, to be scrubbed per rules): EMR Serverless FTA (table read), EMR Serverless FGAC (table read), EMR on EC2 FTA, EMR on EC2 FGAC, Athena v3 FGAC, Glue ETL FGAC, Redshift Provisioned, Redshift Serverless. Each engine/mode in a ```json fenced block, covering both FTA and FGAC where the engine supports both (per spec).

**Note on EMR on EC2 FTA:** the source doc's EMR-on-EC2 FTA subsection is headers-only (no JSON payload); only the FGAC example has a full event. Do **not** fabricate a CloudTrail event. Instead, present the EMR on EC2 FTA subsection with a one-line note that an FTA `GetDataAccess` event for EMR on EC2 is structurally identical to the EMR Serverless FTA example shown above (same `tableArn` + `LakeFormationTrustedCallerInvocation` shape, differing only in the role ARN / `platformType: EMR_ON_EC2`), so the reader is not left thinking the mode is unsupported. This satisfies the spec's "FTA and FGAC" coverage without inventing data.

- [ ] **Step 1: Write intro + scrubbing legend + per-engine examples**

- Intro: what a `GetDataAccess` event is, `eventSource: lakeformation.amazonaws.com`, that it's a management/read-only event emitted when an integrated engine requests credentials/authorization for a table or data location. Link to LF CloudTrail public doc.
- **Scrubbing legend** admonition (`!!! note "About these examples"`) with the placeholder table from the spec; state these are sanitized samples.
- One `### <Engine> — <Mode>` subsection per example above, each with a scrubbed ```json block. For Athena, note FTA is not supported. Keep `platformType`, `requesterService`, and service `sourceIPAddress` values intact.

- [ ] **Step 2: Verify strict build passes**

Run: `source .venv/bin/activate && mkdocs build --strict`
Expected: exit 0.

- [ ] **Step 3: Scrub audit (manual grep)**

Run: `grep -nE '134851836413|hocanint|ASIA|AROAR6ZOLXH|18\.218\.40\.42' content/auditing/lake-formation-cloudtrail.md`
Expected: **no matches** (all real values replaced). If any match, fix before committing.

- [ ] **Step 4: Commit**

```bash
git add content/auditing/lake-formation-cloudtrail.md
git commit -m "Add LF CloudTrail page: GetDataAccess examples per engine"
```

- [ ] **Step 5: Add field-reference table (Identity / Source-engine / Resource)**

Three grouped subsections or one table with a "Group" column. Cover:
- **Identity**: `userIdentity.arn`, `sessionContext.sessionIssuer.arn`, `additionalEventData.lakeFormationPrincipal`, `additionalEventData.lakeFormationRoleSessionName` (call out: session name is the join key to S3 events).
- **Source/engine**: `sourceIPAddress`, `userAgent`, `requesterService`, `requestParameters.auditContext.additionalAuditContext.platformType`, `jobIdentifier`.
- **Resource**: `requestParameters.tableArn` vs `dataLocations`, `permissions`, `supportedPermissionTypes` / `CELL_FILTER_PERMISSION`, `cellLevelSecurityEnforced`. In this subsection explain the **FTA vs FGAC vs direct-S3 signals** (FGAC → `cellLevelSecurityEnforced: true` and/or `CELL_FILTER_PERMISSION`; direct-S3/data-location → `dataLocations` present instead of `tableArn`; `LakeFormationAuthorizedCaller` tag).

- [ ] **Step 6: Add quick-reference engine cheat table + caveats**

- **Cheat table**: columns Engine | `sourceIPAddress` | `userAgent` | `platformType`. Rows for EMR Serverless, EMR on EC2, Athena, Glue ETL, Redshift, Redshift Serverless (values from the examples).
- **Caveats** (`!!! warning`): Table Direct S3 via LF not supported when FGAC enabled on EMR/Glue; EMR/Glue do not support FTA mode with LF for Iceberg tables.

- [ ] **Step 7: Verify strict build passes**

Run: `source .venv/bin/activate && mkdocs build --strict`
Expected: exit 0.

- [ ] **Step 8: Commit**

```bash
git add content/auditing/lake-formation-cloudtrail.md
git commit -m "Add LF CloudTrail page: field guide, cheat table, caveats"
```

---

## Task 4: s3-audit-events.md

**Files:**
- Modify: `content/auditing/s3-audit-events.md`

- [ ] **Step 1: Write the decision-guide content**

- Framing: to see which S3 objects were actually read, you need S3-level auditing; two options.
- **Comparison table**: rows for cost model, delivery latency, level of detail, completeness/guarantees, joins to LF via session name, setup complexity; columns "CloudTrail S3 Data Events" vs "S3 Server Access Logs".
- **Recommendation** (`!!! tip`): CloudTrail S3 Data Events (filtered) as primary for auditability; S3 Server Access Logs as lower-cost/high-volume alternative.
- **Cost optimization for CloudTrail Data Events**: scope event selectors to specific buckets/prefixes and specific operations (`GetObject`/`PutObject`/`DeleteObject`/`ListBucket`) to control volume/cost. Link to advanced event selectors doc.
- **Enabling** subsection: links to public docs for enabling both (S3 CloudTrail data events, S3 server access logging).

- [ ] **Step 2: Verify strict build passes**

Run: `source .venv/bin/activate && mkdocs build --strict`
Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add content/auditing/s3-audit-events.md
git commit -m "Add S3 audit events page: Data Events vs Access Logs decision guide"
```

---

## Task 5: querying-audit-data.md

**Files:**
- Modify: `content/auditing/querying-audit-data.md`

**Source queries** (from the provided doc, scrub account ID in table names to `111122223333`): (1) CloudTrail S3 Data Events joined to LF `GetDataAccess`; (2) S3 Server Access Logs joined to LF `GetDataAccess`.

- [ ] **Step 1: Write intro + both Athena queries at the top**

- Short intro: most customers ETL/stitch S3 access records with LF `GetDataAccess` for end-to-end auditing (scheduled or on-demand); both queries below.
- Note that readers should substitute their own Athena table names (the ones shown use `111122223333` as the example account).
- Two ```sql fenced blocks with the full queries (account IDs scrubbed).

- [ ] **Step 2: Add join-logic explanation + mermaid diagram**

- Prose explaining the join: `s3_sts_session_name` (parsed from the S3 event's assumed-role ARN) = `lakeFormationRoleSessionName` from the LF event; how `access_method` (LF vs DIRECT) and `lf_source` (engine) are derived.
- Mermaid diagram (fenced ```mermaid) showing: `GetDataAccess (LF)` → `AssumeRole session (AWSLF-… session name)` → `S3 GetObject`, and a branch showing direct-S3 access where the session-name linkage is absent.

- [ ] **Step 3: Add limitations admonition + operationalization tips**

- **Limitations** (`!!! warning`): direct S3 access via LF does not populate the LF session name, so those rows cannot be joined back to LF events; note FTA/FGAC caveats affecting log contents.
- **Operationalization tips**: run the join on a schedule (daily/hourly ETL into a curated table) vs on-demand; partition CloudTrail and Access-Log tables by date to control scan cost.

- [ ] **Step 4: Verify strict build passes**

Run: `source .venv/bin/activate && mkdocs build --strict`
Expected: exit 0.

- [ ] **Step 5: Scrub audit (manual grep)**

Run: `grep -nE '134851836413|hocanint' content/auditing/querying-audit-data.md`
Expected: **no matches**.

- [ ] **Step 6: Commit**

```bash
git add content/auditing/querying-audit-data.md
git commit -m "Add querying audit data page: Athena queries, diagram, limitations"
```

---

## Task 6: Final verification

**Files:** none (verification only)

- [ ] **Step 1: Full strict build**

Run: `source .venv/bin/activate && mkdocs build --strict`
Expected: exit 0, zero warnings.

- [ ] **Step 2: Repo-wide scrub audit across the new section**

Run: `grep -rnE '134851836413|hocanint|ASIAR6ZOLXH|AROAR6ZOLXH|18\.218\.40\.42' content/auditing/ content/index.md`
Expected: **no matches**. (The `AROAR6ZOLXH` prefix matches every real principal ID in the source examples.)

- [ ] **Step 3: Confirm nav ordering and mermaid render**

Run: `grep -n "Auditing" mkdocs.yml` and open `site/auditing/querying-audit-data/index.html` to confirm the mermaid block rendered (search for `class="mermaid"`).
Expected: Auditing block present after LF-Tags; mermaid div present.

- [ ] **Step 4: Final commit if any fixups were needed**

```bash
git add -A && git commit -m "Finalize Auditing section" || echo "nothing to commit"
```
