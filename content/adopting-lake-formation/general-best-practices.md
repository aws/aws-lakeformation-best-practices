# General Best Practices

## Lake Formation Service Linked Role vs Customer Managed Roles

We do not recommend using Lake Formation’s Service Linked Role (SLR) in production environments. Although using SLR is provided for convenience, there are few points to consider if you choose it.  You cannot edit the SLR’s policy if needed. Encrypted catalogs are not supported with SLR for cross account sharing. Significantly large number of S3 locations registered with Lake Formation may cause IAM policy limits to be breached. EMR on EC2 does not support SLR registered locations for data access. Lastly, SLR's do not adhere to [AWS Service Control Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html). Hence, we recommend creating a custom IAM role to always register your data locations with Lake Formation. 

## Using Lake Formation Administrator Role

 Lake Formation Administrator permissions could be over-permissive for specific operations on the data lake. There is also a limit of 30 on the number of administrators that can be registered. We recommend you to create roles with limited IAM and Lake Formation permissions for targeted operations. Limit who can be a Lake Formation administrator role to those that need to administer the AWS account and processes that must have full access to the Glue Data Catalog. For read-only access to the entire Glue Data Catalog, we recommend using Read-Only Lake Formation Administrator for operations such as auditing that do not require write access.  For use cases where a user needs to be able to grant access to other users within the account, provide them with Grantable permissions on resources that they manage. 

Below are examples of when not to use Lake Formation administrator:

* Glue crawler role’s - Do not make Glue crawler roles as Lake Formation administrator. Grant permissions to the crawler role to create tables in select databases in the catalog. Grant data location permission to the crawler role on specific S3 locations to which the databases point to. 
* Data producer/owner/steward - Grant data location permission to create databases and tables in specific S3 locations. Provide Grantable permissions to these personas on the databases and tables that they manage.

## When to use Data Filters versus a Glue View ?

A data filter can be created only on individual/specific table while a Glue View can be created across multiple tables. A data filter could be used when you have simple and specific row and column filtering needs on a table. 

The [Glue Views](https://docs.aws.amazon.com/athena/latest/ug/views-glue.html) are different from that of the standard table views that you create in [Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/create-view.html), [Apache Hive](https://docs.aws.amazon.com/athena/latest/ug/hive-views.html), Apache Spark, Presto, Trino, etc. The Glue Views is a feature of Data Catalog in Preview and are based on definer semantics, where access to the Glue Views are defined by the user who creates the view. 

You can share a Glue View without sharing the underlying tables. The recipient can query only the Glue View and not access the tables.  You can share a Glue View from one account to another but cannot create a new Glue View that contains tables shared from other accounts. 

Glue Views is currently in [Preview](https://docs.aws.amazon.com/lake-formation/latest/dg/working-with-views.html). We will add more best practices as the feature becomes generally available. 

