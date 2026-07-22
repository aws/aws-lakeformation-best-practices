# Lake Formation CloudTrail Events

When an integrated analytics engine needs to read a Lake Formation–governed table or data location, it calls Lake Formation's `GetDataAccess` API. Lake Formation evaluates the request against the grants you have defined and, if the request is authorized, vends short-lived credentials scoped to the resource. Every one of these calls is recorded in AWS CloudTrail as a `GetDataAccess` event with `eventSource: lakeformation.amazonaws.com`.

`GetDataAccess` is a **read-only management event** (`readOnly: true`, `managementEvent: true`). It is the authoritative record of *who was authorized to access what, through which engine* — the starting point for any Lake Formation audit trail. See the [Lake Formation CloudTrail documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/logging-using-cloudtrail.html) for how to enable and locate these events.

!!! note "About the examples on this page"

    All events below are **sanitized samples**. Real values have been replaced with placeholders so nothing sensitive is exposed:

    | Real value | Placeholder used |
    | --- | --- |
    | AWS account ID | `111122223333` |
    | S3 bucket name | `amzn-s3-demo-bucket` |
    | Access key ID (`accessKeyId`) | `AKIAIOSFODNN7EXAMPLE` |
    | Principal IDs (`AROA…`) and session suffixes | short example values |
    | Request / event IDs, `x-amz-id-2` | example UUIDs |
    | Query IDs, authorization IDs, table IDs, cluster IDs | example values |
    | Customer IP addresses | `203.0.113.10` (from the [documentation range](https://datatracker.ietf.org/doc/html/rfc5737)) |

    Engine and service identifiers — `sourceIPAddress` service endpoints (for example `athena.amazonaws.com`), `platformType`, and `requesterService` — are kept as-is, because they are documented, non-sensitive, and are exactly the fields you use to identify the calling engine.

## Example events per engine

Each integrated engine emits the same `GetDataAccess` event shape. The differences that matter for auditing are in a handful of fields — the calling identity, the engine signature, and the requested resource — covered in the [field reference](#field-reference) below.

### Amazon EMR Serverless — Full Table Access

In Full Table Access (FTA) mode the request carries a `tableArn` with no cell-filtering metadata. `requesterService` is `UNKNOWN` and the engine is identified from `sourceIPAddress` / `userAgent` and the `platformType` inside `additionalAuditContext`.

```json
{
  "eventVersion": "1.11",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAEXAMPLEEMRS:00exampleJob1,00exampleAttempt1",
    "arn": "arn:aws:sts::111122223333:assumed-role/EmrServerlessRuntimeRole/00exampleJob1,00exampleAttempt1",
    "accountId": "111122223333",
    "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAEXAMPLEEMRS",
        "arn": "arn:aws:iam::111122223333:role/EmrServerlessRuntimeRole",
        "accountId": "111122223333",
        "userName": "EmrServerlessRuntimeRole"
      },
      "attributes": {
        "creationDate": "2026-07-01T22:35:24Z",
        "mfaAuthenticated": "false"
      }
    },
    "invokedBy": "ops.emr-serverless.amazonaws.com"
  },
  "eventTime": "2026-07-01T22:36:08Z",
  "eventSource": "lakeformation.amazonaws.com",
  "eventName": "GetDataAccess",
  "awsRegion": "us-east-2",
  "sourceIPAddress": "ops.emr-serverless.amazonaws.com",
  "userAgent": "ops.emr-serverless.amazonaws.com",
  "requestParameters": {
    "tableArn": "arn:aws:glue:us-east-2:111122223333:table/flights/flights_local",
    "auditContext": {
      "additionalAuditContext": "{\"platformType\":\"EMR_SERVERLESS\",\"jobIdentifier\":\"00exampleJob1/00exampleAttempt1\"}"
    }
  },
  "responseElements": null,
  "additionalEventData": {
    "requesterService": "UNKNOWN",
    "LakeFormationTrustedCallerInvocation": "true",
    "lakeFormationPrincipal": "arn:aws:iam::111122223333:role/EmrServerlessRuntimeRole",
    "lakeFormationRoleSessionName": "AWSLF-00-NA-111122223333-EXAMPLE01"
  },
  "requestID": "aaaaaaaa-1111-2222-3333-444444444444",
  "eventID": "bbbbbbbb-1111-2222-3333-555555555555",
  "readOnly": true,
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "111122223333",
  "sharedEventID": "cccccccc-1111-2222-3333-666666666666",
  "vpcEndpointId": "emr-serverless.amazonaws.com",
  "vpcEndpointAccountId": "emr-serverless.amazonaws.com",
  "eventCategory": "Management"
}
```

### Amazon EMR Serverless — Fine-Grained Access Control

In Fine-Grained Access Control (FGAC) mode the request adds `permissions` (for example `SELECT`), `supportedPermissionTypes: ["CELL_FILTER_PERMISSION"]`, and a `querySessionContext`. `requesterService` is populated (`EMRSERVERLESS`).

```json
{
  "eventVersion": "1.11",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAEXAMPLEFGAC:00exampleJob2,00exampleAttempt2",
    "arn": "arn:aws:sts::111122223333:assumed-role/EmrServerlessRuntimeRole-LF-FGAC/00exampleJob2,00exampleAttempt2",
    "accountId": "111122223333",
    "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAEXAMPLEFGAC",
        "arn": "arn:aws:iam::111122223333:role/EmrServerlessRuntimeRole-LF-FGAC",
        "accountId": "111122223333",
        "userName": "EmrServerlessRuntimeRole-LF-FGAC"
      },
      "attributes": {
        "creationDate": "2026-07-02T14:39:26Z",
        "mfaAuthenticated": "false"
      }
    },
    "invokedBy": "ops.emr-serverless.amazonaws.com"
  },
  "eventTime": "2026-07-02T14:41:04Z",
  "eventSource": "lakeformation.amazonaws.com",
  "eventName": "GetDataAccess",
  "awsRegion": "us-east-2",
  "sourceIPAddress": "ops.emr-serverless.amazonaws.com",
  "userAgent": "ops.emr-serverless.amazonaws.com",
  "requestParameters": {
    "tableArn": "arn:aws:glue:us-east-2:111122223333:table/flights/flights_local",
    "permissions": ["SELECT"],
    "auditContext": {
      "additionalAuditContext": "{\"platformType\":\"EMR_SERVERLESS\",\"jobIdentifier\":\"00exampleJob2/00exampleAttempt2\"}"
    },
    "supportedPermissionTypes": ["CELL_FILTER_PERMISSION"],
    "querySessionContext": {
      "queryId": "11111111-2222-3333-4444-555555555555",
      "queryStartTime": "Jul 2, 2026 2:41:00 PM",
      "queryAuthorizationId": "EXAMPLEQUERYAUTHID01"
    }
  },
  "responseElements": null,
  "additionalEventData": {
    "requesterService": "EMRSERVERLESS",
    "LakeFormationTrustedCallerInvocation": "true",
    "lakeFormationPrincipal": "arn:aws:iam::111122223333:role/EmrServerlessRuntimeRole-LF-FGAC",
    "lakeFormationRoleSessionName": "AWSLF-00-NA-111122223333-EXAMPLE02"
  },
  "requestID": "dddddddd-1111-2222-3333-777777777777",
  "eventID": "eeeeeeee-1111-2222-3333-888888888888",
  "readOnly": true,
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "111122223333",
  "sharedEventID": "ffffffff-1111-2222-3333-999999999999",
  "vpcEndpointId": "emr-serverless.amazonaws.com",
  "vpcEndpointAccountId": "emr-serverless.amazonaws.com",
  "eventCategory": "Management"
}
```

### Amazon EMR on EC2 — Full Table Access

The `GetDataAccess` event for a Full Table Access read on EMR on EC2 is structurally identical to the [EMR Serverless FTA example](#amazon-emr-serverless-full-table-access) above — a `tableArn` request with no cell-filtering metadata and `requesterService: UNKNOWN`. The distinguishing fields are the runtime role ARN and `platformType: EMR_ON_EC2` inside `additionalAuditContext` (see the FGAC example below for the EMR-on-EC2 identity and engine signature).

### Amazon EMR on EC2 — Fine-Grained Access Control

EMR on EC2 calls Lake Formation directly, so `sourceIPAddress` is the cluster's IP and `userAgent` is an AWS SDK string (rather than a service endpoint). FGAC access tagged by EMR also carries `LakeFormationAuthorizedSessionTag: "LakeFormationAuthorizedCaller:Amazon EMR"`.

```json
{
  "eventVersion": "1.11",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAEXAMPLEEC2:5000",
    "arn": "arn:aws:sts::111122223333:assumed-role/human-resource-runtime-role/5000",
    "accountId": "111122223333",
    "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAEXAMPLEEC2",
        "arn": "arn:aws:iam::111122223333:role/human-resource-runtime-role",
        "accountId": "111122223333",
        "userName": "human-resource-runtime-role"
      },
      "attributes": {
        "creationDate": "2026-07-08T17:01:19Z",
        "mfaAuthenticated": "false"
      }
    }
  },
  "eventTime": "2026-07-08T17:02:35Z",
  "eventSource": "lakeformation.amazonaws.com",
  "eventName": "GetDataAccess",
  "awsRegion": "us-east-2",
  "sourceIPAddress": "203.0.113.10",
  "userAgent": "aws-sdk-java/2.42.12 md/io#sync md/http#Apache ua/2.1 api/LakeFormation#2.42.x os/Linux lang/java#17.0.19 md/vendor#Amazon.com_Inc.",
  "requestParameters": {
    "tableArn": "arn:aws:glue:us-east-2:111122223333:table/flights/flights_local",
    "permissions": ["SELECT"],
    "auditContext": {
      "additionalAuditContext": "{\"platformType\":\"EMR_ON_EC2\",\"jobIdentifier\":\"j-EXAMPLECLUSTER/application_1234567890123_0001\"}"
    },
    "supportedPermissionTypes": ["CELL_FILTER_PERMISSION"],
    "querySessionContext": {
      "queryId": "22222222-3333-4444-5555-666666666666",
      "queryStartTime": "Jul 8, 2026 5:02:34 PM",
      "queryAuthorizationId": "EXAMPLEQUERYAUTHID02"
    }
  },
  "responseElements": null,
  "additionalEventData": {
    "requesterService": "UNKNOWN",
    "LakeFormationAuthorizedSessionTag": "LakeFormationAuthorizedCaller:Amazon EMR",
    "LakeFormationTrustedCallerInvocation": "true",
    "lakeFormationPrincipal": "arn:aws:iam::111122223333:role/human-resource-runtime-role",
    "lakeFormationRoleSessionName": "AWSLF-00-NA-111122223333-EXAMPLE03"
  },
  "requestID": "12341234-1111-2222-3333-123412341234",
  "eventID": "56785678-1111-2222-3333-567856785678",
  "readOnly": true,
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "111122223333",
  "eventCategory": "Management",
  "tlsDetails": {
    "tlsVersion": "TLSv1.3",
    "cipherSuite": "TLS_AES_128_GCM_SHA256",
    "clientProvidedHostHeader": "lakeformation.us-east-2.amazonaws.com"
  }
}
```

### Amazon Athena (engine v3) — Fine-Grained Access Control

Athena queries FGAC-enabled tables through Lake Formation; Full Table Access is not supported. Athena events set `requesterService: ATHENA`, `cellLevelSecurityEnforced: true`, and an `expectedTableId`, and carry the Athena `queryId` in `additionalAuditContext`.

Unlike the other engines, Athena writes `additionalAuditContext` as a loosely-formatted string (`{queryId: <id>}`) rather than escaped JSON, so `json_extract_scalar` does not parse it — treat it as an opaque string when querying.

```json
{
  "eventVersion": "1.11",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAEXAMPLEADMIN:admin-session",
    "arn": "arn:aws:sts::111122223333:assumed-role/Admin/admin-session",
    "accountId": "111122223333",
    "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAEXAMPLEADMIN",
        "arn": "arn:aws:iam::111122223333:role/Admin",
        "accountId": "111122223333",
        "userName": "Admin"
      },
      "attributes": {
        "creationDate": "2026-07-01T20:43:33Z",
        "mfaAuthenticated": "false"
      }
    },
    "invokedBy": "athena.amazonaws.com"
  },
  "eventTime": "2026-07-01T21:01:47Z",
  "eventSource": "lakeformation.amazonaws.com",
  "eventName": "GetDataAccess",
  "awsRegion": "us-east-2",
  "sourceIPAddress": "athena.amazonaws.com",
  "userAgent": "athena.amazonaws.com",
  "requestParameters": {
    "tableArn": "arn:aws:glue:us-east-2:111122223333:table/flights/flights_local",
    "permissions": ["SELECT"],
    "auditContext": {
      "additionalAuditContext": "{queryId: 33333333-4444-5555-6666-777777777777}"
    },
    "cellLevelSecurityEnforced": true,
    "expectedTableId": "EXAMPLETABLEID0000000000000000"
  },
  "responseElements": null,
  "additionalEventData": {
    "requesterService": "ATHENA",
    "LakeFormationTrustedCallerInvocation": "true",
    "lakeFormationPrincipal": "arn:aws:iam::111122223333:role/Admin",
    "lakeFormationRoleSessionName": "AWSLF-00-AT-111122223333-EXAMPLE04"
  },
  "requestID": "43214321-1111-2222-3333-432143214321",
  "eventID": "87658765-1111-2222-3333-876587658765",
  "readOnly": true,
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "111122223333",
  "eventCategory": "Management"
}
```

### AWS Glue ETL — Fine-Grained Access Control

Glue ETL jobs (Glue 5.1+) access FGAC tables through Lake Formation with `platformType: GLUE_ETL`. As with EMR Serverless, `requesterService` is `UNKNOWN` and the engine is identified from `sourceIPAddress` / `userAgent` (`glue.amazonaws.com`) and `platformType`.

```json
{
  "eventVersion": "1.11",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAEXAMPLEGLUE:GlueJobRunnerSession",
    "arn": "arn:aws:sts::111122223333:assumed-role/GlueServiceRole-NoS3/GlueJobRunnerSession",
    "accountId": "111122223333",
    "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAEXAMPLEGLUE",
        "arn": "arn:aws:iam::111122223333:role/GlueServiceRole-NoS3",
        "accountId": "111122223333",
        "userName": "GlueServiceRole-NoS3"
      },
      "attributes": {
        "creationDate": "2026-07-09T17:20:10Z",
        "mfaAuthenticated": "false"
      }
    },
    "invokedBy": "glue.amazonaws.com"
  },
  "eventTime": "2026-07-09T17:24:08Z",
  "eventSource": "lakeformation.amazonaws.com",
  "eventName": "GetDataAccess",
  "awsRegion": "us-east-2",
  "sourceIPAddress": "glue.amazonaws.com",
  "userAgent": "glue.amazonaws.com",
  "requestParameters": {
    "tableArn": "arn:aws:glue:us-east-2:111122223333:table/flights/flights_local",
    "permissions": ["SELECT"],
    "auditContext": {
      "additionalAuditContext": "{\"platformType\":\"GLUE_ETL\",\"jobIdentifier\":\"EXAMPLE-glue-job-run-id/\"}"
    },
    "supportedPermissionTypes": ["CELL_FILTER_PERMISSION"],
    "querySessionContext": {
      "queryId": "44444444-5555-6666-7777-888888888888",
      "queryStartTime": "Jul 9, 2026 5:24:05 PM",
      "queryAuthorizationId": "EXAMPLEQUERYAUTHID03"
    }
  },
  "responseElements": null,
  "additionalEventData": {
    "requesterService": "UNKNOWN",
    "LakeFormationTrustedCallerInvocation": "true",
    "lakeFormationPrincipal": "arn:aws:iam::111122223333:role/GlueServiceRole-NoS3",
    "lakeFormationRoleSessionName": "AWSLF-00-NA-111122223333-EXAMPLE05"
  },
  "requestID": "9a9a9a9a-1111-2222-3333-9a9a9a9a9a9a",
  "eventID": "9b9b9b9b-1111-2222-3333-9b9b9b9b9b9b",
  "readOnly": true,
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "111122223333",
  "eventCategory": "Management"
}
```

### Amazon Redshift (provisioned)

Redshift Spectrum requests carry `durationSeconds`, `cellLevelSecurityEnforced`, and `expectedTableId` (no `platformType`). `requesterService` is `REDSHIFT`, and console-driven sessions include `sessionCredentialFromConsole: "true"`.

```json
{
  "eventVersion": "1.11",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAEXAMPLEADMIN:admin-session",
    "arn": "arn:aws:sts::111122223333:assumed-role/Admin/admin-session",
    "accountId": "111122223333",
    "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAEXAMPLEADMIN",
        "arn": "arn:aws:iam::111122223333:role/Admin",
        "accountId": "111122223333",
        "userName": "Admin"
      },
      "attributes": {
        "creationDate": "2026-07-08T15:12:27Z",
        "mfaAuthenticated": "false"
      }
    },
    "invokedBy": "redshift.amazonaws.com"
  },
  "eventTime": "2026-07-08T22:01:13Z",
  "eventSource": "lakeformation.amazonaws.com",
  "eventName": "GetDataAccess",
  "awsRegion": "us-east-2",
  "sourceIPAddress": "redshift.amazonaws.com",
  "userAgent": "redshift.amazonaws.com",
  "requestParameters": {
    "tableArn": "arn:aws:glue:us-east-2:111122223333:table/flights/flights_local",
    "durationSeconds": 3600,
    "cellLevelSecurityEnforced": true,
    "expectedTableId": "EXAMPLETABLEID0000000000000000"
  },
  "responseElements": null,
  "additionalEventData": {
    "requesterService": "REDSHIFT",
    "LakeFormationTrustedCallerInvocation": "true",
    "lakeFormationPrincipal": "arn:aws:iam::111122223333:role/Admin",
    "lakeFormationRoleSessionName": "AWSLF-00-RE-111122223333-EXAMPLE06"
  },
  "requestID": "5c5c5c5c-1111-2222-3333-5c5c5c5c5c5c",
  "eventID": "5d5d5d5d-1111-2222-3333-5d5d5d5d5d5d",
  "readOnly": true,
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "111122223333",
  "eventCategory": "Management",
  "sessionCredentialFromConsole": "true"
}
```

### Amazon Redshift Serverless

Redshift Serverless is identified by `sourceIPAddress` / `userAgent` of `redshift-serverless.amazonaws.com`. Its `additionalAuditContext` embeds the invoking Redshift database user rather than a `platformType`.

```json
{
  "eventVersion": "1.11",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAEXAMPLEADMIN:admin-session",
    "arn": "arn:aws:sts::111122223333:assumed-role/Admin/admin-session",
    "accountId": "111122223333",
    "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAEXAMPLEADMIN",
        "arn": "arn:aws:iam::111122223333:role/Admin",
        "accountId": "111122223333",
        "userName": "Admin"
      },
      "attributes": {
        "creationDate": "2026-07-08T15:12:27Z",
        "mfaAuthenticated": "false"
      }
    },
    "invokedBy": "redshift-serverless.amazonaws.com"
  },
  "eventTime": "2026-07-08T21:19:12Z",
  "eventSource": "lakeformation.amazonaws.com",
  "eventName": "GetDataAccess",
  "awsRegion": "us-east-2",
  "sourceIPAddress": "redshift-serverless.amazonaws.com",
  "userAgent": "redshift-serverless.amazonaws.com",
  "requestParameters": {
    "tableArn": "arn:aws:glue:us-east-2:111122223333:table/flights/flights_local",
    "durationSeconds": 3600,
    "auditContext": {
      "additionalAuditContext": "{\"invokedBy\":\"arn:aws:redshift:us-east-2:111122223333:dbuser:serverless-111122223333-EXAMPLE/IAMR:Admin\",\"transactionId\":\"3309\",\"queryId\":\"NULL\",\"isConcurrencyScalingQuery\":\"false\"}"
    },
    "cellLevelSecurityEnforced": true,
    "expectedTableId": "EXAMPLETABLEID0000000000000000"
  },
  "responseElements": null,
  "additionalEventData": {
    "requesterService": "UNKNOWN",
    "LakeFormationTrustedCallerInvocation": "true",
    "lakeFormationPrincipal": "arn:aws:iam::111122223333:role/Admin",
    "lakeFormationRoleSessionName": "AWSLF-00-RE-111122223333-EXAMPLE07"
  },
  "requestID": "5e5e5e5e-1111-2222-3333-5e5e5e5e5e5e",
  "eventID": "5f5f5f5f-1111-2222-3333-5f5f5f5f5f5f",
  "readOnly": true,
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "111122223333",
  "eventCategory": "Management",
  "sessionCredentialFromConsole": "true"
}
```

## Field reference

The fields most useful for auditing fall into three groups: **who** made the request (identity), **what engine** made it (source/engine), and **what** they were authorized to access (resource).

### Identity fields

| Field | What it tells you |
| --- | --- |
| `userIdentity.arn` | The assumed-role session ARN that called `GetDataAccess` (includes the session name suffix). |
| `userIdentity.sessionContext.sessionIssuer.arn` | The underlying IAM role — the stable identity to group activity by, regardless of session. |
| `additionalEventData.lakeFormationPrincipal` | The IAM principal Lake Formation evaluated grants against. This is the principal whose permissions authorized the access. |
| `additionalEventData.lakeFormationRoleSessionName` | The session name (for example `AWSLF-00-AT-111122223333-EXAMPLE04`) of the short-lived credential Lake Formation vends. **This is the key that links a `GetDataAccess` event to the S3 object accesses that follow it** — see [Querying Audit Data](querying-audit-data.md). |

### Source / engine fields

| Field | What it tells you |
| --- | --- |
| `sourceIPAddress` / `userAgent` | For managed engines these are a service endpoint (for example `athena.amazonaws.com`); for EMR on EC2 the source is the cluster IP and the agent is an AWS SDK string. |
| `additionalEventData.requesterService` | The calling service when populated (`ATHENA`, `REDSHIFT`, `EMRSERVERLESS`). Often `UNKNOWN` for EMR Serverless FTA, EMR on EC2, Glue ETL, and Redshift Serverless — fall back to the fields below. |
| `requestParameters.auditContext.additionalAuditContext.platformType` | The engine type where present (`EMR_SERVERLESS`, `EMR_ON_EC2`, `GLUE_ETL`). |
| `additionalAuditContext.jobIdentifier` | The job/application identifier of the caller — ties the access to a specific EMR or Glue job run. |
| `requestParameters.querySessionContext.queryId` / `queryAuthorizationId` | The engine's query identifiers, for correlating back to a specific query. |

### Resource fields

| Field | What it tells you |
| --- | --- |
| `requestParameters.tableArn` | The Glue Data Catalog table being accessed (table-level request). |
| `requestParameters.dataLocations` | Present instead of `tableArn` when access is requested by S3 data location rather than by table. |
| `requestParameters.permissions` | The permissions being exercised, for example `SELECT`. |
| `requestParameters.supportedPermissionTypes` | Includes `CELL_FILTER_PERMISSION` when the request is subject to fine-grained (row/column/cell) filtering. |
| `requestParameters.cellLevelSecurityEnforced` | `true` when cell-level security was enforced for the access (seen from Athena and Redshift). |

#### Telling FTA, FGAC, and direct-S3 access apart

The resource fields also reveal which access mode was used:

- **Fine-Grained Access Control (FGAC):** the event carries `permissions` (for example `["SELECT"]`) together with `cellLevelSecurityEnforced: true` and/or `supportedPermissionTypes: ["CELL_FILTER_PERMISSION"]`, usually alongside a populated `requesterService` and `querySessionContext`.
- **Full Table Access (FTA):** the event has a `tableArn` (with `auditContext`) but **no** `permissions` and **no** cell-filtering metadata; `requesterService` is typically `UNKNOWN`.
- **Access by data location (direct S3):** `requestParameters.dataLocations` is present instead of `tableArn`. Note that these requests do **not** populate `lakeFormationRoleSessionName`, which limits how they can be correlated to S3 events (see the limitations in [Querying Audit Data](querying-audit-data.md)).
- EMR-tagged FGAC access additionally carries `additionalEventData.LakeFormationAuthorizedSessionTag: "LakeFormationAuthorizedCaller:Amazon EMR"`.

## Quick reference: identifying the engine

Use this table to identify which engine produced a `GetDataAccess` event from its signature fields.

| Engine | `sourceIPAddress` / `userAgent` | `platformType` | Other signals |
| --- | --- | --- | --- |
| EMR Serverless | `ops.emr-serverless.amazonaws.com` | `EMR_SERVERLESS` | `requesterService` `UNKNOWN` (FTA) or `EMRSERVERLESS` (FGAC) |
| EMR on EC2 | Cluster IP / `aws-sdk-java/…` | `EMR_ON_EC2` | `LakeFormationAuthorizedSessionTag: LakeFormationAuthorizedCaller:Amazon EMR` |
| Athena (v3) | `athena.amazonaws.com` | *(none)* | `requesterService` `ATHENA`; `queryId` in `additionalAuditContext`; FGAC only |
| Glue ETL | `glue.amazonaws.com` | `GLUE_ETL` | `requesterService` `UNKNOWN` |
| Redshift (provisioned) | `redshift.amazonaws.com` | *(none)* | `requesterService` `REDSHIFT`; `durationSeconds` present |
| Redshift Serverless | `redshift-serverless.amazonaws.com` | *(none)* | `additionalAuditContext.invokedBy` names the Redshift `dbuser` |

## Caveats

!!! warning "Access-mode limitations on EMR and Glue"

    - Table access directly by S3 data location through Lake Formation is **not supported** when FGAC is enabled on EMR or Glue.
    - EMR and Glue do **not** support Full Table Access mode with Lake Formation for Apache Iceberg tables.
