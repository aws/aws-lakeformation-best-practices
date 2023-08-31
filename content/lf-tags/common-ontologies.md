# Common LF-Tag Ontologies

Here is a list of common LF-Tags that make up a tagging ontology that are commonly seen by customers:

1. Environment (eg. dev, beta, gamma, prod) - LF-Tag that denotes which stage a particular resource exists and granting developers to resource in dev and beta, but only application access to gamma and prod.

2. Departments (eg. sales, marketing, engineering, etc) - LF-Tag that groups department datasets using a single tag. This can be used for sharing purpose (ie departmentA shares with departmentB).

3. Product (eg. dataproduct1, dataproduct2, etc) - LF-Tag that groups similar datasets under a data product that be used for permissioning and sharing.
4. Owner (eg. GroupA, GroupB, GroupC) - LF-Tag that provides certain groups of principals the ability different set of permissions (like write/delete permissions) that is separate for sharing purposes. 

5. Roles (eg. data scientist, data engineer, application, etc) - LF-Tag that provides different levels of permissions based on role. For example, a data steward could have grantable full access to a dataset where as a data scientist could have read only permissions.

6. Data Classification (eg. SSN, Address, PhoneNumber, etc ) - LF-Tag that allows to permission differently based on the classification of data. More sensitive classifications can be accessable by privileged consumers, etc.

7. Data Sensitivity (eg. public, private, PII, etc) - LF-Tag that allows different groups of principals to accessible columns or tables based on their sensitivity of the column to meet compliance rules, like GDPR. 

8. Sharable (eg. sharable, notsharable) - LF-Tag that groups resources that can be shared with other departments.  
 