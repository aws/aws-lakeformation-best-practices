# Updating Lake Formation cross account sharing versions

Lake Formation improves its sharing features over time, which sometimes requires changes that are not backwards compatible. To give customers flexibility in choosing when to upgrade, Lake Formation uses versioned sharing modes. Each new version retains the benefits of previous versions while adding enhancements.

To maximize the benefits of cross-account sharing, it is recommended to use the newest version of cross-account sharing (currently Version 4) when first setting up Lake Formation permissions or when upgrading from older versions. 

## Cross account sharing version 1

This version creates an AWS Resource Access Manager (RAM) resource share for every cross-account Lake Formation permission granted through the named resource method. If there are many shares, this can exceed limits on the number of resource shares. For Lake Formation tag-based data sharing, cross-account permissions don't use AWS RAM but they rely on Data Catalog resource policy. 

To avoid hitting any limits, it is best to upgrade to a newer version at your earliest convenience.

## Cross account sharing version 2

This version streamlines cross-account permission grants between two AWS accounts by using a single AWS RAM resource share to represent multiple grants. This mapping dramatically reduces the number of required resource shares compared to individually sharing each permission grant, making cross-account sharing more scalable. While named resources use AWS RAM for cross-account sharing, LF-Tags based sharing still relies on Data Catalog resource policies. Lake Formation allows sharing catalog resources at the external account level but not at the IAM principal level.

If your system utilizes LF-Tags for sharing data, it is strongly advised that you upgrade to a more recent version.

## Cross account sharing version 3

This version streamlines cross-account resource sharing by using AWS RAM to map multiple cross-account permissions to a single resource share. Optimization of AWS RAM resource shares reduces the number of shares needed for cross-account access.  Further, this version allows sharing resources directly to external account IAM principals as well as sharing at the account level. Additionally, LF-Tags based access control would use AWS RAM for sharing, eliminating the need to update Glue Data Catalog resource policies, hence streamlining sharing with named-resources.

## Cross account sharing version 4

This version includes the benefits of V3. Additionally, it enables resources to be managed in Hybrid Access Mode and also allows sharing hybrid resources across accounts. For more details refer to [documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/hybrid-access-mode.html)


# FAQs

1. What are the considerations to be met before the migration?
 

We have summarized the recommended cross account sharing versions for a source and target account combination. 

                             | Target account(receiver) V1 | Target account(receiver) V2 | Target account(receiver) V3 | Target account(receiver) - V4 
-----------------------------| ----------------------------|-----------------------------|-----------------------------|-------------------------------                           
Source Account(grantor) - V1 |	yes                        | yes 	                     | not possible                | not possible
Source Account(grantor) - V2 |	yes                        | yes                         | not possible                | not possible
Source Account(grantor) - V3 |  yes                        | yes                         |  yes                        | yes 
Source Account(grantor) - V4 |  yes                        | yes                         | 	yes 	                   | yes 
				

2. When my source account's cross account version is updated to V3/V4, will existing resources shared to othere accounts that are using  V1 or V2 continue working?

Yes, updating the cross account version settings in your source account does not modify the permissions that the recipient has on shared resources. You do not need to revoke any existing Lake Formation permissions. However, you may consider revoking and re-granting cross account permissions if you are using V1 sharing and are hitting your RAM resource share limits. Revoking and re-granting will consolidate multiple RAM resource shares into fewer resource shares and give the scalability to share more resources.


## Steps to be taken in source account

1. The IAM role or IAM user granting the permission (that is, the grantor)must have the policies mentioned in the AWS managed policy arn:aws:iam::aws:policy/AWSLakeFormationCrossAccountManager for granting the cross-account permissions. You can also attach the managed policy to your role. 

2. If you currently use Glue catalog resource policy,  add the following permission as well to the resource policy.  Replace <grantor_account_id> with the AWS account ID of the grantor account, and <region> with the AWS region where your resources are located. If you do not have an existing Glue catalog resource policy, no additional steps are required.

```json
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
```
3. Update the cross account version of the source or grantor account to V4.

Once you choose  Version 4, all new named resource grants will go through the new cross-account grant mode when using Lake Formation sharing. 

To optimally use AWS RAM capacity for your existing cross-account shares, you can follow the below steps:

1. Get the list of permissions defined in the account using list-permissions API, and filter out the cross-account permissions on resources owned by the account. In case of named resource sharing you can filter the permissions that have ResourceShare listed under AdditionalDetails. For more details refer to [documentation](https://docs.aws.amazon.com/cli/latest/reference/lakeformation/list-permissions.html)

Example output of list-permissions for a cross account share.
```json
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
```
2. Revoke the permission granted on resources to cross account principal: This removes the V1 RAM share corresponding to the resources.

3. Re-Grant the permission on resources: This establishes V4 RAM share for the resources shared. 

4. Optionally you can validate the resources that are shared using RAM invites by invoking [RAM API](https://docs.aws.amazon.com/cli/latest/reference/ram/) for named resource sharing.

CAUTION in step 2: Once access is revoked for the external account from source account, any principals in the target account who have cascaded access will lose their access to the resource.

NOTE in step 3: You can also revoke or grant permissions using batch APIs for each database and its tables that are shared across accounts incrementally, and validate if the shared resource is accessible in the target account. For more information refer to [documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/cross-account-notes.html)

NOTE: Permissions granted on the LF-Tags to cross account will stay in tact.


## Steps to be taken in target account

1. To receive and accept resource shares using AWS RAM, data lake administrators in target accounts must have an additional policy that grants permission to accept AWS RAM resource share invitations and enable resource sharing with organizations. For information on how to enable sharing with organizations, see [Enable sharing with AWS organizations](https://docs.aws.amazon.com/ram/latest/userguide/getting-started-sharing.html#getting-started-sharing-orgs) in the AWS RAM User Guide.
```json
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
```
2. If the source and target accounts are part of an AWS Organization and if resource sharing is enabled, then the accounts within that Organization will automatically have access to the shared resources without needing RAM invitations. 
3. When resource sharing is  not enabled, for resources shared with target account, target account should see the new RAM invitations. Data lake admin needs to accept these RAM invites. This establishes Glue catalog resource policies for target account to access the shared resource.
4. In case of re-grant of existing shared resource from source account(grantor) with new cross account version, existing resource links and permissions on those resource links should be intact for account admin and other principals in the target account(receiver).
5. Validate access to the shared resource in the target account.


Note:
In order to query a shared database or table using Amazon Athena in the recipient account,  you need a resource link.  In the case where the database or table is shared to a direct IAM principal in the recipient account, the principal needs Lake Formation CREATE_TABLE or CREATE_DATABASE permission to create the resource link. They also need  the glue:CreateTable or glue:CreateDatabase IAM permission in their IAM policy (based on resource type that is shared). 

For more information on this topic, refer to the [blog](https://aws.amazon.com/blogs/big-data/introducing-hybrid-access-mode-for-aws-glue-data-catalog-to-secure-access-using-aws-lake-formation-and-iam-and-amazon-s3-policies/)


## Other References

For details on Lake Formation cross account sharing, refer [documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/cross-account-permissions.html)

For more details on cross account sharing versions, refer [documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/optimize-ram.html). 

