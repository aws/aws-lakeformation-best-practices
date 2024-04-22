# Cross Account Versions

Lake Formation improves its sharing features over time, which sometimes requires changes that are not backwards compatible. To give customers flexibility in choosing when to upgrade, Lake Formation uses versioned sharing modes. Each new version retains the benefits of previous versions while adding enhancements.

To maximize the benefits of cross-account sharing, it is recommended to use the newest version of cross-account sharing (currently Version 4) when first setting up Lake Formation permissions or when upgrading from older versions.

## Version 1:

This version creates an AWS Resource Access Manager (RAM) resource share for every cross-account Lake Formation permission granted through the named resource method. If there are many shares, this can exceed limits on the number of resource shares. For Lake Formation tag-based data sharing, cross-account permissions don't use AWS RAM but they rely on Data Catalog resource policy. 

To avoid hitting any limits, it is best to upgrade to a newer version at your earliest convenience.

## Version 2:

This version streamlines cross-account permission grants between two AWS accounts by using a single AWS RAM resource share to represent multiple grants. This mapping dramatically reduces the number of required resource shares compared to individually sharing each permission grant, making cross-account sharing more scalable. While named resources use AWS RAM for cross-account sharing, LF-Tags based sharing still relies on Data Catalog resource policies. Lake Formation allows sharing catalog resources at the external account level but not at the IAM principal level.

If your system utilizes LF-Tags for sharing data, it is strongly advised that you upgrade to a more recent version.

## Version 3:

This version streamlines cross-account resource sharing by using AWS RAM to map multiple cross-account permissions to a single resource share. Optimization of AWS RAM resource shares reduces the number of shares needed for cross-account access.  Further, this version allows sharing resources directly to external account IAM principals as well as sharing at the account level. Additionally, LF-Tags based access control would use AWS RAM for sharing, eliminating the need to update Glue Data Catalog resource policies, hence streamlining sharing with named-resources. You can also establish Tag Based cross-account sharing to Organizations or Organizational units.

## Version 4:

This version includes the benefits of V3. Additionally, it enables resources to be managed in Hybrid Access Mode and also allows sharing hybrid resources across accounts.

For more details refer to [public documentation](https://docs.aws.amazon.com/lake-formation/latest/dg/hybrid-access-mode.html)

