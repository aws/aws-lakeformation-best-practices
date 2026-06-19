# Migrating from Apache Ranger to Lake Formation

Organizations running Apache Hadoop clusters often rely on Apache Ranger to manage fine-grained access control over data stored in HDFS, Hive Metastore, and related services. As these organizations migrate analytics workloads to AWS, they encounter AWS Lake Formation, which provides column-level and row-level permissions for data in Amazon S3 registered through the AWS Glue Data Catalog.

Ranger and Lake Formation solve the same core problem — controlling who can read which tables, columns, and rows — but they differ in architecture, policy model, and identity integration. A Ranger policy cannot be copied into Lake Formation unchanged.

This guide covers:

- [The structural differences between Ranger and Lake Formation](#differences-between-apache-ranger-and-lake-formation)
- [Catalog migration (HMS to Glue Data Catalog)](#hive-metastore-vs-glue-data-catalog)
- [Handling deny and except policies](#deny-and-except-policies)
- [Tag-based access control translation](#tag-based-access-control)
- [Security zones and multi-tenancy](#security-zones)
- [Identity mapping from LDAP/AD to IAM](#identity-mapping)
- [HDFS to S3](#hdfs-to-s3)

> **Note:** This guide covers the access-control layer only. It does not cover migrating data itself (ETL pipelines, schema conversion, S3 layout).

---

## Differences Between Apache Ranger and Lake Formation

Apache Ranger and AWS Lake Formation both govern access to analytical data, but they differ in where enforcement happens, how policies are expressed, and what identity systems they consume.

| Dimension | Apache Ranger | AWS Lake Formation |
|-----------|---------------|-------------------|
| **Policy model** | Allow, deny, and exceptions. Deny takes precedence over allow. | Grant-only. No deny grants exist. Permissions are additive; the union of all grants determines effective access. |
| **Permission granularity** | Database, table, column, row-level (row-filter policies), and cell-level (column masking + row filter). | Database, table, column, row-level (data filters), and cell-level (column + row filter combined in a data filter). |
| **Catalog system** | Hive Metastore (HMS). | AWS Glue Data Catalog (GDC). All governed resources must exist in GDC. |
| **Identity integration** | LDAP, Active Directory, Kerberos principals, OS users/groups. | IAM principals (roles, users), IAM Identity Center users/groups, and AWS Organizations. |
| **Tag-based access control** | Atlas-sourced or Ranger-native tags. A single tag policy covers all resources sharing that tag across services. | LF-Tags (key-value pairs) assigned to databases, tables, and columns in GDC. Boolean expressions over tag keys in a single grant. |
| **Multi-tenancy / security zones** | Security zones partition the resource namespace with delegated administrators per zone. | No direct equivalent. Isolation is achieved through separate AWS accounts, IAM role delegation with grantable permissions, or distinct LF-Tag taxonomies. |
| **Auditing** | Ranger audit logs in Solr or HDFS. Each access decision recorded with user, resource, policy ID, and timestamp. | AWS CloudTrail logs Lake Formation API calls and data access events for each vended credential. |
| **Enforcement point** | Plugin-per-service model. Each service (Hive, Spark, HBase, HDFS) loads a Ranger plugin that evaluates policies locally. | Centralized. Lake Formation vends temporary credentials scoped to the granted permissions. S3 enforces access via those scoped credentials. |

---

## Hive Metastore vs. Glue Data Catalog

Apache Ranger can enforce policies against resources in either a Hive Metastore or the AWS Glue Data Catalog. Lake Formation operates exclusively against the Glue Data Catalog. Every database, table, and column that Lake Formation governs must exist in GDC.

This creates a hard dependency in migration sequencing: **catalog migration to GDC must complete before policy translation can begin.** Ranger policies reference catalog resources by name (database, table, column). If those resources do not yet exist in GDC, there is nothing for a Lake Formation grant to bind to.

For detailed steps on migrating Hive Metastore metadata into the Glue Data Catalog, refer to the [HMS to Glue Data Catalog migration guide](https://aws.highspot.com/items/6a284d07987cca17721d8f59).

### Migration action

Complete the Hive Metastore to Glue Data Catalog migration before beginning policy translation. Validate that all databases, tables, and partitions referenced by existing Ranger policies are present in GDC. Any resource missing from GDC will be an orphaned policy with no equivalent grant target in Lake Formation.

---

## Deny and Except Policies

Apache Ranger evaluates access decisions using a three-layer model: allow policies, deny policies, and exceptions to each. When a request arrives, Ranger checks:

1. Is there a **deny policy** that matches this user and resource? If yes, deny — unless the user falls under an exception to that deny (an "exclude from deny" entry).
2. Is there an **allow policy** that matches? If yes, allow — unless the user falls under an exception to that allow (an "exclude from allow" entry).
3. If no policy matches, deny by default.

This model lets administrators write broad allow rules and carve out specific exclusions without enumerating every permitted user. For example: "All analysts can query the `customers` table, except the `ssn` column" is a single allow policy on `customers.*` with an exclude-from-allow on `customers.ssn`.

**Lake Formation has no deny mechanism.** Grants are purely additive. The effective permissions for a principal are the union of all grants that reference that principal. There is no way to subtract a permission once granted.

### Worked example

**Ranger policy:** Allow group "analysts" SELECT on table `customers`, all columns. Exclude from allow: group "analysts" on column `customers.ssn`.

**Lake Formation equivalent:** Grant SELECT on table `customers`, listing every column *except* `ssn` explicitly to the IAM role or Identity Center group representing "analysts."

This works but introduces a maintenance cost: any new column added to `customers` must be explicitly added to the grant. In Ranger, the wildcard allow plus column-level exclude handled this automatically.

### Workarounds by pattern

| Ranger pattern | Lake Formation approach |
|----------------|------------------------|
| Allow all columns, deny specific columns | Grant SELECT on included columns only (explicit column list). Update grants when schema changes. |
| Allow all users, deny specific users | Do not grant the excluded users. Since grants are additive, omission is the equivalent of denial. |
| Allow a group, exclude a subgroup | Create separate IAM roles or Identity Center groups that reflect the split. Grant only to the permitted subset. |
| Row-level deny (deny rows matching a filter) | Use data filters with `RowFilter` expressions that return only permitted rows. Invert the deny condition into an allow condition. |

### Patterns with no clean equivalent

- **Priority-ordered deny:** In Ranger, a deny policy with higher priority overrides a lower-priority allow. Lake Formation has no policy priority concept. The union of all grants determines access.
- **Dynamic exclusion:** Ranger deny policies can use runtime attributes (request context, time-of-day via custom conditions) to deny access dynamically. Lake Formation grants are static.
- **Cross-table deny with wildcards:** A single Ranger deny policy can reference multiple tables using wildcards (e.g., deny group "interns" access to all tables matching `*_pii`). Lake Formation grants are per-resource; use LF-Tags to approximate wildcard coverage across tables.

### Migration action

Inventory all Ranger deny and exclude-from-allow policies. For each one, determine whether the equivalent restriction can be achieved by narrowing the corresponding Lake Formation grant (granting only the permitted subset). Where the deny pattern relies on wildcards, priority ordering, or runtime conditions, the access model will need restructuring — typically by splitting principals into more granular groups or restructuring table layouts to separate sensitive columns into distinct tables.

---

## Tag-Based Access Control

Apache Ranger supports tag-based policies through integration with Apache Atlas (or Ranger's own tag service). Atlas classifies resources (tables, columns) with tags such as "PII", "CONFIDENTIAL", or "FINANCE". Ranger policies then reference these tags rather than specific resource names. A single tag policy controls access to every resource carrying a given tag, regardless of which database or table it belongs to. Ranger tag policies follow the same allow/deny/exception model as resource policies.

AWS Lake Formation provides **LF-Tags**: key-value pairs assigned to databases, tables, and columns in the Glue Data Catalog. Permissions are granted using tag expressions (e.g., grant SELECT to a principal on all resources where `sensitivity=high` AND `domain=finance`). LF-Tags support inheritance: a tag assigned to a database propagates to all tables and columns within it unless overridden at a lower level.

### Key differences

| Aspect | Ranger (Atlas tags) | Lake Formation (LF-Tags) |
|--------|---------------------|--------------------------|
| **Tag source** | Apache Atlas classification, or Ranger native tag service | Assigned directly in Glue Data Catalog via LF API |
| **Tag structure** | Flat classification names (e.g., "PII"), optionally with attributes | Key-value pairs (e.g., `sensitivity=high`) with enumerated allowed values per key |
| **Policy model on tags** | Allow, deny, and exceptions on tagged resources | Grant-only on tag expressions (no deny) |
| **Scope** | Cross-service (Hive, HBase, HDFS, Kafka; any Atlas-classified resource) | Glue Data Catalog resources only (databases, tables, columns) |
| **Inheritance** | Determined by Atlas lineage and propagation rules | Hierarchical: database → table → column, with override at each level |
| **Compound expressions** | One tag per policy; multiple policies combine via evaluation order | Boolean AND expressions over multiple tag keys in a single grant |

### Migration steps

1. **Inventory Atlas tags in use.** Extract all tag classifications referenced by Ranger tag policies. Map each to an LF-Tag key-value pair. Flat classifications like "PII" become a key-value such as `sensitivity=PII` or `classification=PII`.

2. **Define the LF-Tag taxonomy.** LF-Tags require predefined keys and allowed values. Unlike Atlas, where any string can be a tag, LF-Tag values must be enumerated upfront. Plan the taxonomy before assigning tags.

3. **Assign LF-Tags to catalog resources.** For each resource that carried an Atlas tag, apply the corresponding LF-Tag in GDC. Use inheritance (tag the database) where the tag applies uniformly; override at the table or column level where exceptions exist.

4. **Translate tag policies to LF-Tag grants.** Each Ranger tag-allow policy becomes a Lake Formation tag-based grant. Where Ranger tag-deny policies exist, the same constraints from the [deny/except section](#deny-and-except-policies) apply: denial must be restructured as omission from the grant or separation of principals.

5. **Account for scope reduction.** Ranger tag policies may cover non-catalog resources (HDFS paths, Kafka topics, HBase tables). These are outside Lake Formation's scope. Access control for those resources must be handled separately (S3 bucket policies, MSK IAM auth, or other AWS-native mechanisms).

---

## Security Zones

Apache Ranger security zones partition the resource namespace into isolated administrative domains. Each zone defines a set of resources (specific databases, tables, or path prefixes) and assigns zone-level administrators who can create and manage policies only within their zone's boundary. A resource belongs to exactly one zone.

This model supports multi-tenant Hadoop environments where separate teams manage their own access policies without interfering with each other.

**Lake Formation has no direct equivalent to security zones.** There is no built-in mechanism to partition the Glue Data Catalog into named administrative scopes within a single AWS account.

### Achieving similar isolation

| Ranger security zone pattern | Lake Formation approach |
|------------------------------|------------------------|
| Zone per business unit with delegated admin | Create an IAM role per zone. Grant that role grantable permissions (`GRANT OPTION`) on the specific databases and tables belonging to that zone. The role can administer access within its delegated resources without affecting resources outside its scope. |
| Zone scoped to specific databases | Grant grantable permissions on those databases and tables to the zone-admin role. That role can grant downstream access to other principals, but only on resources it holds grantable permissions for. |
| Hard trust boundaries between zones | Separate AWS accounts per boundary. Each account has its own Glue Data Catalog and Lake Formation administrators. Cross-account resource links provide controlled sharing where needed. |

### Migration action

Inventory all Ranger security zones and the resources assigned to each. For each zone, create a corresponding IAM role to serve as the zone administrator. Grant that role grantable permissions on the zone's resources in Lake Formation. Principals who previously administered policies within a Ranger zone now assume the corresponding IAM role to manage grants within their delegated scope.

Where zones represent hard trust boundaries — teams that must not be able to affect each other's access under any circumstance — use separate AWS accounts rather than role-based delegation within a single account.

---

## Identity Mapping

Apache Ranger policies bind to identities sourced from external directory services. In a typical Hadoop deployment, Ranger resolves users and groups from LDAP, Active Directory, or Unix/OS-level user stores, with Kerberos providing authentication. A Ranger policy might grant access to `group:data-engineers` or `user:jsmith`, where those names correspond to entries in the connected directory.

Lake Formation grants bind to AWS identity primitives:

- **IAM users and roles:** grants reference ARNs (e.g., `arn:aws:iam::123456789012:role/analytics-role`)
- **IAM Identity Center users and groups:** grants reference Identity Center ARNs
- **AWS Organizations:** grants can target an entire organization or organizational unit for cross-account sharing

The identity that Lake Formation evaluates depends on how the environment is configured. In the default model, Lake Formation authorizes based on the IAM principal (role or user) making the API call. When **Trusted Identity Propagation (TIP)** is enabled, Lake Formation authorizes based on the IAM Identity Center user identity instead — the actual end-user identity is passed through the query engine to Lake Formation, bypassing the IAM role as the authorization subject.

### Key differences

| Aspect | Apache Ranger | Lake Formation |
|--------|---------------|----------------|
| **Identity source** | LDAP, Active Directory, Unix users, Kerberos principals | IAM roles/users, IAM Identity Center |
| **Group model** | Directory groups (nested groups supported depending on directory) | IAM Identity Center groups, or implicit grouping via shared IAM roles |
| **Policy binding** | Username or group name string match | IAM principal ARN (default), or Identity Center user/group identity (with Trusted Identity Propagation) |
| **Authentication** | Kerberos (typically), or delegated to the service | AWS SigV4 (IAM credentials or Identity Center session) |
| **User-to-principal mapping** | 1:1 — user authenticates as themselves | Many-to-one possible when using IAM roles; 1:1 when using TIP with Identity Center |

### Migration action

Map every Ranger user and group referenced in existing policies to an AWS identity. The approach depends on the organization's identity strategy:

1. **Identity Center with Trusted Identity Propagation (recommended for named-user mapping):** Sync the existing LDAP/AD directory to IAM Identity Center via SCIM or AD Connector. Users and groups retain their names in Identity Center. Enable Trusted Identity Propagation so Lake Formation authorizes based on the Identity Center identity directly. This preserves the per-user, per-group access model from Ranger most closely.

2. **IAM roles per functional group:** Where organizations use role-based access (all analysts assume an "analyst" role), create IAM roles corresponding to each Ranger group. Grant Lake Formation permissions to those roles. Users authenticate via Identity Center and assume the appropriate role.

3. **Handling unmapped identities:** Ranger policies referencing service accounts, OS-level users, or Kerberos principals with no corresponding AWS identity require new IAM roles or Identity Center accounts to be provisioned. These often represent service-to-service access that translates to IAM role trust policies rather than Lake Formation grants.

Extract the full list of users and groups from Ranger's UserSync or the underlying directory. Cross-reference against the set of principals that will exist in AWS post-migration. Any gap — a Ranger identity with no AWS equivalent — must be resolved before policies can be translated.

---

## HDFS to S3

Apache Ranger provides path-level access control for HDFS. Ranger HDFS policies grant or deny read, write, and execute permissions on directory paths (e.g., allow group "etl-team" write access to `/data/warehouse/finance/`). These policies operate at the storage layer, independent of any catalog or table abstraction.

AWS Lake Formation does not govern storage-path access. Lake Formation permissions apply only to catalog resources (databases, tables, columns) registered in the Glue Data Catalog. When a workload accesses S3 directly by path rather than through a catalog-aware query engine, Lake Formation is not involved in the authorization decision.

For organizations that relied on Ranger HDFS policies to control who can read or write specific S3-prefixed paths, **Amazon S3 Access Grants** fills this gap.

S3 Access Grants provide location-based grants that map S3 prefixes to IAM or Identity Center identities. An S3 Access Grants instance defines locations (S3 prefixes such as `s3://my-bucket/finance/`) and grants access to those locations for specific principals at a defined permission level (READ, WRITE, or READWRITE).

### Comparison

| Aspect | Ranger HDFS policies | S3 Access Grants |
|--------|---------------------|-----------------|
| **Target** | HDFS directory paths | S3 bucket prefixes |
| **Permissions** | Read, write, execute (with deny/exception support) | READ, WRITE, READWRITE (grant-only, no deny) |
| **Identity** | LDAP/AD users and groups | IAM principals or IAM Identity Center users/groups |
| **Granularity** | Any path depth in HDFS | Any prefix depth in S3 |
| **Scope** | Per-cluster (each cluster has its own Ranger) | Per S3 Access Grants instance (can span multiple buckets within a region) |
| **Integration with catalog permissions** | Independent of Hive/Ranger resource policies (complementary layer) | Independent of Lake Formation catalog grants (complementary layer) |

### Migration guidance

1. **Inventory Ranger HDFS policies.** Extract all path-based policies and the principals they reference. Identify which HDFS paths correspond to S3 locations in the migrated environment.

2. **Map HDFS paths to S3 prefixes.** For each HDFS path governed by a Ranger policy, determine the equivalent S3 prefix in the target architecture. Example: `/data/warehouse/finance/` becomes `s3://datalake-bucket/warehouse/finance/`.

3. **Create an S3 Access Grants instance.** Register the S3 locations (prefixes) that need path-level access control. Associate the instance with an Identity Center instance if using named-user grants.

4. **Translate Ranger HDFS grants to S3 Access Grants.** Each Ranger HDFS allow policy becomes an S3 Access Grant at the corresponding prefix. Since S3 Access Grants has no deny model, the same constraints from the deny/except section apply: restrict access by granting only the permitted subset of principals.

5. **Complement Lake Formation with S3 Access Grants.** Use Lake Formation for catalog-level permissions (table/column/row access) and S3 Access Grants for storage-level permissions (direct path access). The two systems are complementary: Lake Formation governs query engines reading through the Glue Data Catalog, while S3 Access Grants governs direct S3 API access to the underlying data files.

---

## How to Get Help

- **AWS Lake Formation documentation** at [docs.aws.amazon.com/lake-formation](https://docs.aws.amazon.com/lake-formation/) covers grant management, LF-Tags, data filters, Trusted Identity Propagation, and cross-account sharing.
- **AWS Prescriptive Guidance:** Published migration patterns and architecture recommendations for analytics workloads moving to AWS-native services.
- **AWS Solutions Architects:** Customers with an AWS account team can engage their Solutions Architect for architecture reviews of their migration plan, particularly for complex deny-policy translation and multi-account design.
- **AWS re:Post:** Community Q&A for specific implementation questions related to Lake Formation permissions and Glue Data Catalog configuration.
- **AWS Support:** For customers with Business or Enterprise support plans, support cases can be opened for Lake Formation configuration and troubleshooting.
