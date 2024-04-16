# Upgrade Cross Account Version 
Below table provides summarize of various combinations for source and target account's cross account versions that is supported.

| Source account\Target account    | Target account(receiver) - V1 | Target account(receiver) - V2| Target account(receiver) - V3| Target account(receiver) - V4|
| -------- | ------- | -------- | -------- | -------- |
| Source Account(grantor) - V1  | supported    | supported    | not supported    | not supported    |
| Source Account(grantor) - V2  | supported   | supported    | not supported    | not supported    |
| Source Account(grantor) - V3  | supported    | supported    | supported    | supported   |
| Source Account(grantor) - V4  | supported    | supported    | supported    | supported   |

###  Note:
Modifying the grantor account's cross-account version settings does not change existing permissions the receiver account has on shared resources. However, if you are using V1 sharing and hitting your RAM resource share limit, consider revoking and re-granting cross-account permissions in the source account. Doing so will consolidate multiple RAM resource shares into fewer shares, allowing you to share more resources.

## Steps to be taken in Source Account:

1. The IAM role or IAM user granting the permission (that is, the grantor) must have the policies mentioned in the AWS managed policy arn:aws:iam::aws:policy/AWSLakeFormationCrossAccountManager for granting the cross-account permissions. You can also attach the managed policy to your role. 
2. If you currently use Glue catalog resource policy, add the following permission as well to the resource policy.  Replace <grantor_account_id> with the AWS account ID of the grantor account, and <region> with the AWS region where your resources are located. If you do not have an existing Glue catalog resource policy, no additional steps are required.

{
    "Version": "2012-10-17",
    "Statement": [
        {
           "Effect": "Allow",
            "Action": [
                "glue:ShareResource"
            ],
            "Principal": {"Service": [
                "ram.amazonaws.com"
            ]},
            "Resource": [
                "arn:aws:glue:<region>:<grantor_account_id>:catalog",
                "arn:aws:glue:<region>:<grantor_account_id>:database/*",
                "arn:aws:glue:<region>:<grantor_account_id>:table/*"
            ]
        }
    ]
}


1. Update the cross-account version of the source or grantor account to V4.

Once you choose Version 4, all new named resource grants will go through the new cross-account grant mode when using Lake Formation sharing. 

Optionally to optimize AWS RAM usage for your existing cross-account shares, you can follow the below steps:

1. Get the list of permissions defined in the account using list-permissions API, and filter out the cross-account permissions on resources owned by the account. In case of named resource sharing you can filter the permissions that have ResourceShare listed under AdditionalDetails. For more details refer to https://docs.aws.amazon.com/cli/latest/reference/lakeformation/list-permissions.html

Example output of list-permissions for a cross account share.

````

{
    "Principal": {
        "DataLakePrincipalIdentifier": "987654321012"
    },
    "Resource": {
        "Database": {
            "CatalogId": "123456789012",
            "Name": "salesdb"
        }
    },
    "Permissions": [
        "DESCRIBE"
    ],
    "PermissionsWithGrantOption": [
        "DESCRIBE"
    ],
    "AdditionalDetails": {
        "ResourceShare": [
            "arn:aws:ram:us-east-1:123456789012:resource-share/15bc4e61-1423-4c44-9452-c23fda161f3f"
        ]
    },
    "LastUpdated": "2024-03-14T19:07:25.687000+00:00",
    "LastUpdatedBy": "arn:aws:iam::123456789012:role/LFAdmin"
}
````

1. Revoke the permission granted on resources to cross account principal: This removes the V1 RAM share corresponding to the resources. 
    Caution: Once access is revoked for the external account from source account, any principals in the target account who have cascaded access will lose their access to the resource. 
    
2. Re-Grant the permission on resources: This establishes V4 RAM share for the resources shared. 

Note: You can also revoke or grant permissions using batch APIs for each database and its tables that are shared across accounts incrementally, and validate if the shared resource is accessible in the target account. For more information refer to https://docs.aws.amazon.com/lake-formation/latest/dg/cross-account-notes.html

1. Optionally you can validate the resources that are shared using RAM invites by invoking RAM API for named resource sharing. https://docs.aws.amazon.com/cli/latest/reference/ram/

Note: Permissions granted on the LF-Tags to cross account will stay in tact.

## Step to be taken in Target Account:

1. To receive and accept resource shares using AWS RAM, data lake administrators in target accounts must have an additional policy that grants permission to accept AWS RAM resource share invitations and enable resource sharing with organizations. For information on how to enable sharing with organizations, see Enable sharing with AWS organizations in the AWS RAM User Guide.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ram:AcceptResourceShareInvitation",
                "ram:RejectResourceShareInvitation",
                "ec2:DescribeAvailabilityZones",
                "ram:EnableSharingWithAwsOrganization"
            ],
            "Resource": "*"
        }
    ]
}
````

1. If the source and target accounts are part of an AWS Organization and if resource sharing is enabled, then the accounts within that Organization will automatically have access to the shared resources without needing RAM invitations. 
2. When resource sharing is not enabled, for resources shared with target account, target account should see the new RAM invitations. Data lake admin needs to accept these RAM invites. This establishes Glue catalog resource policies for target account to access the shared resource.
3. In case of re-grant of existing shared resource from source account(grantor) with new cross account version, existing resource links and permissions on those resource links should be intact for account admin and other principals in the target account(receiver).
4. Validate access to the shared resource in the target account.

### Note:
In order to query a shared database or table using Amazon Athena in the recipient account, you need a resource link.  In the case where the database or table is shared to a direct IAM principal in the recipient account, the principal needs Lake Formation CREATE_TABLE or CREATE_DATABASE permission to create the resource link. They also need the glue:CreateTable or glue:CreateDatabase IAM permission in their IAM policy (based on resource type that is shared). 
For more information on this topic, refer to the blog: https://aws.amazon.com/blogs/big-data/introducing-hybrid-access-mode-for-aws-glue-data-catalog-to-secure-access-using-aws-lake-formation-and-iam-and-amazon-s3-policies/

## Other References:

https://docs.aws.amazon.com/lake-formation/latest/dg/cross-account-permissions.html



