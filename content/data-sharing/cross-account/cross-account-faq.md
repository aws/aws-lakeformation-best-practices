# Cross Account FAQS

### 1. What are the various cross account topologies customers can share resources?

Lake formation cross account sharing feature is used for sharing catalog and underlying data lake access. Customer can share resource without duplicating data avoiding extensive pipelines to make the data available to other lines of business. These models enable various lines of business collaboration and require planning to standardize the enterprise level adaption.

#### Central governance model:

This specialized form of a hub and spoke model is chosen when customers want to centralizes metadata discovery and permission management in governance account and make the data available to other AWS accounts. In this model domain accounts will share the data with central governance account that hosts the metadata for enterprises to discovery and request access for the resource without domain owners involved in the fulfilment process. Central governance account provides better audibility on resource access and scaling the permission across enterprise.  

For more details on the central governance model setup refer to blog: 
https://aws.amazon.com/blogs/big-data/introducing-aws-glue-crawlers-using-aws-lake-formation-permission-management/

#### Peer to Peer:

Peer to Peer model is one of the data mesh patterns that is used by customers who have high degree of autonomy established within the domain and wants to operate in decentralized fashion.  In this model, domain accounts act as data producers and the data ownership remains with the domain generating them. Domain owners of the data will manage the lifecycle of the data and also control access to the data when sharing it with other domains. These domains operate independently and don’t depend on central governance platform/team for data distribution/fulfillment. 

For more details on the setup refer to blog:
https://aws.amazon.com/blogs/big-data/securely-share-your-data-across-aws-accounts-using-aws-lake-formation/

### 2. What is various cross account sharing versions available in Lake Formation?

See [Cross Account versions] (cross-account-versions.md) for various cross account versions available.

### 3. What are the factors to consider when choosing account level sharing vs direct sharing with principal?

|     | Account to Account | Account to Principal| 
| -------- | ------- | -------- | 
|Control model  | When a source account (producer) wants to share a resource with a target account (consumer) but expects the consumer's data lake administrator to manage the resource permissions within their account, the source account can grant access at the account level rather than specifying individual users.    | When source account(producer) wants to have tighter control on resource permission and wants to completely control who have access to the resource, customers can choose sharing the resource to direct principal in target account.    | 
| Cross-Account version requirement | All Cross Account version supports this model   | Cross Account version V3 and above supports this model    | 
| Receiver policies | Since the resource is shared at the account level, the administrator of the target account's data lake should have policies to create a resource link and delegate permissions. No additional policy requirements exist for other principals that are delegated permissions to the resource in the target account. For various personas and policies please refer to: https://docs.aws.amazon.com/lake-formation/latest/dg/permissions-reference.html#lf-permissions-tables    | Since the resource is shared cross account to principal, each receiver principal the needs Lake Formation CREATE_TABLE or CREATE_DATABASE permission to create the resource link. They also need the glue:CreateTable or glue:CreateDatabase IAM permission in their IAM policy (based on resource type that is shared).    | 
| Auditability  | To understand who has access to a given dataset, you need to review the permissions defined in both the source account where the dataset originated and the target account with which the dataset has been shared.    | To understand who has access to a given dataset, you can determine this information entirely from the source account.    | 

For more details on direct sharing with principal, refer to blog:
 https://aws.amazon.com/blogs/big-data/enable-cross-account-sharing-with-direct-iam-principals-using-aws-lake-formation-tags/

### 4. Can I share both my Glue Data catalog resources (catalog, database, table) and Redshift tables using Lake Formation cross account.

Yes, you can share both data lake resource cataloged in Glue Data Catalog with storage on S3 and Redshift tables shared with Glue Data catalog via data shares (https://docs.aws.amazon.com/redshift/latest/dg/lf_datashare_overview.html) using Lake Formation for cross account access.

 For more details refer to blog:
https://aws.amazon.com/blogs/big-data/implement-tag-based-access-control-for-your-data-lake-and-amazon-redshift-data-sharing-with-aws-lake-formation/

### 5. Can I share my resource cross account cross region?

yes, Lake Formation allows querying Data Catalog tables across regions using Athena, EMR, and Glue ETL. By creating resource links in other regions pointing to source databases and tables, you can access data across regions without copying the underlying data or metadata into the Data Catalog. Engines that query the data needs network connectivity to endpoints within the region to access S3 buckets. For example, EMR clusters or Glue ETL jobs running in a private subnet within an AWS VPC may require a NAT gateway, VPC peering, or transit gateway to reach external resources. Any network traffic between source instance to any AWS endpoint stays within the AWS network and does not go over the internet. Refer  to https://aws.amazon.com/vpc/faqs/ for details on network traffic and communication path.

 For more details on cross region setup refer to blog:
https://aws.amazon.com/blogs/big-data/configure-cross-region-table-access-with-the-aws-glue-catalog-and-aws-lake-formation/

### 6. Can I share resource cross account using Lake Formation when resource management within that account is not done using Lake Formation?

Yes, you can use Hybrid access Mode to register the S3 bucket containing your data in Lake Formation in the source account(grantor) and opt-in cross account(target) principal for using Lake Formation mode for sharing. This enables target(receiver) account access to the shared resource using Lake Formation.

For more details refer on how hybrid access mode works, refer to:
https://aws.amazon.com/blogs/big-data/introducing-hybrid-access-mode-for-aws-glue-data-catalog-to-secure-access-using-aws-lake-formation-and-iam-and-amazon-s3-policies/


## Other References

For details on Lake Formation cross account sharing, refer [documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/cross-account-permissions.html)

For more details on cross account sharing versions, refer [documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/optimize-ram.html). 
