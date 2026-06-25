# Adopting Lake Formation

AWS Lake Formation centralizes permissions management for your data lake, enabling fine-grained access control over databases, tables, and columns registered in the AWS Glue Data Catalog. This section covers everything you need to plan and execute a Lake Formation adoption — from initial evaluation through migration.

---

## Before You Start

[Considerations when adopting Lake Formation](considerations-when-adopting-lake-formation.md) walks through a structured evaluation checklist covering:

- Taking inventory of your data stores and analytics engines
- Reviewing your current access management model
- Choosing a permissions model (named-resource vs. LF-Tags)
- Understanding identity integration options (IAM, IAM Identity Center, Trusted Identity Propagation)
- Deciding on account topology and cross-account sharing

This is the recommended starting point if you are evaluating Lake Formation for the first time.

---

## Adoption Modes

[Lake Formation Adoption Modes](lake-formation-adoption-modes.md) explains the two modes for registering S3 locations with Lake Formation and when to use each:

- **Lake Formation mode** — Lake Formation is the sole permissions model for governed resources. Recommended for greenfield environments or teams with minimal existing IAM-based access.
- **Hybrid access mode** — Lake Formation and IAM resource-based policies coexist on the same resources. Individual principals can be opted in to Lake Formation while others continue using IAM without interruption. Recommended when existing workloads cannot be migrated all at once.

Understanding adoption modes is a prerequisite for choosing a migration approach.

---

## Migrating to Lake Formation

The right migration path depends on where you are starting from:

| Current Permission Model | Guide |
|--------------------------|-------|
| IAM and S3 bucket policies (AWS-native) | [Lake Formation Transition Guide — Single Account](transition-guide/lake-formation-transition-guide-single-account.md) |
| Apache Ranger (Hadoop / on-premises) | [Migrating from Apache Ranger](transition-guide/apache-ranger-to-lake-formation.md) |

### Transition guides

- **[Lake Formation Transition Guide — Single Account](transition-guide/lake-formation-transition-guide-single-account.md)** — two incremental migration plans for environments already on AWS. Both are reversible and designed for production environments with active consumers.
  - [Plan #1 — Per Resource Transition](transition-guide/single-account-transition-plan-1-per-resource.md): migrate table by table; all consumers of a resource move at once.
  - [Plan #2 — Per User Transition](transition-guide/single-account-transition-plan-2-per-user.md): migrate user by user using Hybrid access mode opt-ins; surgical rollback per user.

- **[Migrating from Apache Ranger](transition-guide/apache-ranger-to-lake-formation.md)** — covers policy model translation, handling Ranger deny/except semantics, tag-based access control mapping, security zone equivalents, and LDAP/AD to IAM identity mapping.

---

## General Best Practices

[General Best Practices](general-best-practices.md) covers configuration recommendations that apply regardless of migration path:

- Using a customer-managed IAM role for data location registration instead of the Lake Formation Service Linked Role
- Scoping Lake Formation Administrator access appropriately
- When to use Data Filters vs. Glue Views for row and column filtering
