# S3 Audit Events

[Lake Formation CloudTrail Events](lake-formation-cloudtrail.md) record the *authorization* side of the story — for example, a `GetDataAccess` event captures the fact that Lake Formation vended temporary credentials to a principal for a given data location. However, these events do not show which S3 objects were subsequently read or written using those credentials. To close that gap you need S3-level auditing. AWS provides two mechanisms: **CloudTrail S3 Data Events** and **S3 Server Access Logs**. This page helps you choose between them.

## Comparison

| Attribute | CloudTrail S3 Data Events | S3 Server Access Logs |
|---|---|---|
| **Cost model** | Billed per event delivered; scoping with advanced event selectors reduces volume and cost | Free to enable; you pay S3 storage for the log files, which can be high-volume |
| **Delivery latency** | Typically within minutes | Best-effort; can take hours |
| **Level of detail** | Rich structured JSON — full IAM identity including the assumed-role session ARN, request parameters, and response elements | Space-delimited fields including requester, operation, object key, HTTP status code, and bytes transferred |
| **Completeness** | Delivered on a best-effort basis but generally complete for enabled event selectors | Best-effort with no delivery guarantee; logs are delivered periodically |
| **Join to Lake Formation via session name** | The assumed-role session name (e.g., `AWSLF-…`) appears in the `userIdentity` ARN, enabling correlation with `GetDataAccess` events | The assumed-role session name appears in the Requester field, enabling the same correlation when the request was made under the LF-vended session |
| **Setup complexity** | Configure a trail with advanced event selectors targeting the relevant buckets or prefixes | Enable server access logging on the source bucket, directing logs to a separate target bucket |

> **Note on session-name correlation:** The join to Lake Formation events works when S3 access is performed under the credentials vended by Lake Formation. In that case the assumed-role session name begins with `AWSLF-` and is the key shared between the Lake Formation `GetDataAccess` event and the S3 audit record.

## Recommendation

!!! tip
    Use **CloudTrail S3 Data Events scoped with advanced event selectors** as the primary auditing mechanism. Limiting selectors to the S3 locations registered with Lake Formation, and to the specific operations you need (`GetObject`, `PutObject`, `DeleteObject`), keeps costs manageable while giving you near-real-time, structured, complete records that correlate cleanly with Lake Formation events.

    **S3 Server Access Logs** are a suitable lower-cost alternative for very high-volume buckets where per-event CloudTrail charges would be prohibitive. Accept the trade-offs: logs are best-effort, can be delayed by hours, and the space-delimited format requires more parsing work. They are not a substitute for CloudTrail when you need reliable, timely audit trails.

## Controlling CloudTrail data event cost

CloudTrail data events are billed by volume, so scoping them carefully is important in data lake environments where S3 activity can be high.

Use **advanced event selectors** to control what is captured:

- **Limit to relevant resources** — specify the S3 bucket ARNs (or ARN prefixes) for buckets registered with Lake Formation. There is no reason to capture events for buckets outside your governed data lake.
- **Limit to relevant operations** — include `GetObject` and `PutObject` for read/write auditing, and `DeleteObject` if deletion auditing is required. Add `ListBucket` (via management read events) only if bucket-listing visibility is needed.
- **Use `readOnly` / `writeOnly` filters** — the `readOnly` field in an advanced event selector lets you capture only read events or only write events, avoiding double-billing for mixed workloads where you need only one direction.
- **Narrow by key prefix** — if your registered data locations use a consistent prefix structure (for example `s3://my-data-lake/governed/`), scope the `resources.ARN` condition to that prefix.

For the selector syntax, see [CloudTrail advanced event selectors](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-data-events-with-advanced-event-selectors.html).

## Enabling S3 auditing

- **CloudTrail S3 data events** — [Enable CloudTrail logging for S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/enable-cloudtrail-logging-for-s3.html)
- **CloudTrail advanced event selectors** (for filtering and cost control) — [Logging data events with advanced event selectors](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-data-events-with-advanced-event-selectors.html)
- **S3 Server Access Logging** — [Amazon S3 server access logging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html)

---

To correlate these S3 records with Lake Formation authorization events, see [Querying Audit Data](querying-audit-data.md).
