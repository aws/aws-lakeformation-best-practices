# Cross Account FAQS

## 1. What are the various cross account topologies customers can share resources?

Lake formation cross account sharing feature is used for sharing catalog and underlying data lake access. Customer can share resource without duplicating data avoiding extensive pipelines to make the data available to other lines of business. These models enable various lines of business collaboration and require planning to standardize the enterprise level adaption.

### Peer to Peer (Pure Data Mesh)

Peer to Peer model is one of the data mesh patterns that is used by customers who have high degree of autonomy established within the domain and wants to operate in decentralized fashion.  In this model, data producers own data products and the data ownership remains with the data producer generating them. Data product owners will manage the lifecycle of the data and also control access to the data when sharing it with consumers. Data producers and data consumers operate independently and don’t depend on central governance platform/team for data distribution/fulfillment. There is a lot of information that we have left out, but if you wish to learn more about data meshes, a great resource is [Data Mesh Principles and Logical Architecture by Zhamak Dehghani](https://martinfowler.com/articles/data-mesh-principles.html)

To see how to build peer to peer data meshes, refer to blog:
[Securely share your data across AWS accounts using AWS Lake Formation](https://aws.amazon.com/blogs/big-data/securely-share-your-data-across-aws-accounts-using-aws-lake-formation/). 

### Hub and Spoke

This topology can be generalized as there is a central AWS account (hub) that always owns the catalog. Then there are nodes that interact with the central account (spokes). However, there are specific implementations where there are two choices: 1/ who owns data sets and 2/ who permissions access to those data sets.

When looking at the decision on who owns the data sets, there are two possibilities: 1/ the central account owns the data, and data producers write their data to the central accounts or 2/ the producers down the data in their own accounts. When the central account owns all the data within the hub and spoke, there can be very strict controls on who can access it, and what data is written. For example, a central team can continously monitor the data to ensure that PII (personally identifiable information) is properly cataloged and secured. When producers own their own data, they have more freedom to do what they want with that data.

When looking at the decision on who governs the datasets, there are two possibilities: 1/ the central account decides whether a consumer has the ability to consume a data set by a central governance team, or 2/ data producers can make those decisions. 

Based on whether the two decisions, customers can achieve different business goals. Some customers choose that all data is stored in the central account, and all decisions on who is allowed to access a dataset is governed centrally. This provides the greatest amount of control on the participants within the organization. Some customers have increasingly decided to decentralize as much as possible by having producers own their datasets, and make decisions on who can access their datasets, similar to how data meshes operate like in our next section.

### Centrally governed data mesh

This a hybrid of both hub and spoke model and data mesh. Customers chose this topology when customers want to centralizes metadata discovery and permission management in governance account and make the data available to other AWS accounts. In this model producers publish their data assets with a central governance account that hosts the metadata for organizations to discovery and request access for the resource. Producers, like in data meshes, will permission consumers. Central governance account provides better audibility on resource access and scaling the permission across enterprise, while proper compliance controls can take place. 

Like data meshes, this requires orgnaizations to use a different paradigm that some organizations may not be used to. However, if this model is attractive, AWS highly recommend that you take a look at [AWS Data Zone](https://aws.amazon.com/datazone/), which leverages Lake Formation for security management, Glue Data Catalog for techincal data management, while offering a business catalog, approval workflows, and Interoperability with third party vendors. 

However, if you wish to implement this model without Data Zone, please refer to blog [Design a data mesh architecture using AWS Lake Formation and AWS Glue](https://aws.amazon.com/blogs/big-data/design-a-data-mesh-architecture-using-aws-lake-formation-and-aws-glue/)

## 2. What are the various cross account sharing versions available in Lake Formation?

See [Cross Account versions](cross-account-versions.md) for various cross account versions available.


## 3. How does Lake Formation manage the AWS RAM resource shares across cross account sharing versions? 

In version 1,  one RAM share is created for every catalog object shared between two accounts. For example, if account A shares database1 with DESCRIBE permission to account B, one RAM share is created. Next, if account A shares table1 from database1, a second RAM share is created. When table2 from database1 is shared, a third RAM share is created. If account A shares database1.ALL_Tables, then a 4th RAM share is created and so on. If account A shares a second database2 with account B, a new RAM share is created. As many Lake Formation cross account grants per account per Data Catalog object between two accounts, as many RAM shares are created. 

In version 2, RAM shares are optimized and continues to be optimized in version 3 and version 4.  A set of RAM shares are created on the source account for every recipient account.  Between version 2 and version 3 (or version 4), the number of RAM shares created are the same when named resource method is used for cross account sharing and it differs slightly for LF-TBAC. 

One RAM share is created in the source account for all databases shared with a recipient account. This database level RAM share gets reused by attaching all new databases shared to the same recipient account. One RAM share is created in the source account for all tables shared with a recipient account. This table level RAM share gets reused by attaching all new tables shared to the same recipient account.  If the source account chooses to share a database with the ALL_Tables option, then a new RAM share is created. So, using the named resource method for cross account grants, a total of up to 3 RAM shares can be created for every account pair. If the source account does not choose to grant cross account permissions using ALL_Tables, then only 2 RAM shares are created for every account pair.

For LF-Tags based cross account sharing, two additional RAM shares are created - one for databases and one for tables. Any additional LF-Tag based grants between the same two accounts reuses these 2 RAM shares.

So, if you are using a combination of LF-TBAC and named resource method for cross account sharing, version 3 and version 4 can create up to 5 RAM shares in the source account for any two source and recipient account pairs.


## 4. What are the factors to consider when choosing account level sharing vs direct sharing with principal?

|     | Account to Account | Account to Principal| 
| -------- | ------- | -------- | 
|Control model  | When a source account (producer) wants to share a resource with a target account (consumer) but expects the consumer's data lake administrator to manage the resource permissions within their account, the source account can grant access at the account level rather than specifying individual users.    | When source account(producer) wants to have tighter control on resource permission and wants to completely control who have access to the resource, customers can choose sharing the resource to direct principal in target account.    | 
| Cross-Account version requirement | All Cross Account version supports this model   | Cross Account version V3 and above supports this model    | 
| Receiver policies | Since the resource is shared at the account level, the administrator of the target account's data lake should have policies to create a resource link and delegate permissions. No additional policy requirements exist for other principals that are delegated permissions to the resource in the target account. For various personas and policies please refer to [public documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/permissions-reference.html#lf-permissions-tables)    | Since the resource is shared cross account to principal, each receiver principal the needs Lake Formation CREATE_TABLE or CREATE_DATABASE permission to create the resource link. They also need the glue:CreateTable or glue:CreateDatabase IAM permission in their IAM policy (based on resource type that is shared).    | 
| Auditability  | To understand who has access to a given dataset, you need to review the permissions defined in both the source account where the dataset originated and the target account with which the dataset has been shared.    | To understand who has access to a given dataset, you can determine this information entirely from the source account.    | 

For more details on direct sharing with principal, refer to blog [Enable cross-account sharing with direct IAM principals using AWS Lake Formation Tags](https://aws.amazon.com/blogs/big-data/enable-cross-account-sharing-with-direct-iam-principals-using-aws-lake-formation-tags/)

## 5. Can I share both my Glue Data catalog resources (catalog, database, table) and Redshift tables using Lake Formation cross account.

Yes, you can share both data lake resource cataloged in Glue Data Catalog with storage on S3 and Redshift tables shared with Glue Data catalog via data shares (https://docs.aws.amazon.com/redshift/latest/dg/lf_datashare_overview.html) using Lake Formation for cross account access.

 For more details refer to blog [Implement tag-based access control for your data lake and Amazon Redshift data sharing with AWS Lake Formation](https://aws.amazon.com/blogs/big-data/implement-tag-based-access-control-for-your-data-lake-and-amazon-redshift-data-sharing-with-aws-lake-formation/)

## 6. Can I share my resource cross account cross region?

yes, Lake Formation allows querying Data Catalog tables across regions using Athena, EMR, and Glue ETL. By creating resource links in other regions pointing to source databases and tables, you can access data across regions without copying the underlying data or metadata into the Data Catalog. Engines that query the data needs network connectivity to endpoints within the region to access S3 buckets. For example, EMR clusters or Glue ETL jobs running in a private subnet within an AWS VPC may require a NAT gateway, VPC peering, or transit gateway to reach external resources. Any network traffic between source instance to any AWS endpoint stays within the AWS network and does not go over the internet. Refer to (AWS VPC FAQs)[https://aws.amazon.com/vpc/faqs/] for details on network traffic and communication path.

 For more details on cross region setup refer to blog [Configure cross-Region table access with the AWS Glue Catalog and AWS Lake Formation](https://aws.amazon.com/blogs/big-data/configure-cross-region-table-access-with-the-aws-glue-catalog-and-aws-lake-formation/)

## 7. Can I share resource cross account using Lake Formation when resource management within that account is not done using Lake Formation?

Yes, you can use Hybrid access Mode to register the S3 bucket containing your data in Lake Formation in the source account(grantor) and opt-in cross account(target) principal for using Lake Formation mode for sharing. This enables target(receiver) account access to the shared resource using Lake Formation.

For more details refer on how hybrid access mode works, refer to blog [Introducing hybrid access mode for AWS Glue Data Catalog to secure access using AWS Lake Formation and IAM and Amazon S3 policies](https://aws.amazon.com/blogs/big-data/introducing-hybrid-access-mode-for-aws-glue-data-catalog-to-secure-access-using-aws-lake-formation-and-iam-and-amazon-s3-policies/)


## Other References

- For details on Lake Formation cross account sharing, refer [public documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/cross-account-permissions.html)
- For more details on cross account sharing versions, refer [public documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/optimize-ram.html). 
