# Securing access to Azure Data Lake Gen 2 from Azure Databricks

**Summary:**

Thisdocument attempts to provide customers of Azure Data Lake (ADLS) and Azure Databricks with an introduction of the common approaches in which data can be accessed and secured in ADLS from Databricks.

**Versions:**

| **Name** | **Title** | **Notes** | **Date** |
| --- | --- | --- | --- |
| Nicholas Hurt | Microsoft Cloud Solution Architect – Data &amp; AI | Original | 20 Jan 2020 |
|   |   |   |   |
|   |   |   |   |

** **





# Contents

Introduction

Pattern 1. Access via Service Principal

Pattern 2. Multiple workspaces — permission by workspace

Pattern 3 — AAD Credential passthrough

Pattern 4. Cluster scoped Service Principal

Pattern 5. Session scoped Service Principal

Pattern 6. Databricks Table Access Control

Conclusion

License/Terms of Use



# Introduction

There are a number of ways to configure access to Azure Data Lake Storage gen2 (ADLS) from Azure Databricks (ADB). This white paper attempts to cover the common patterns, advantages and disadvantages of each, and the scenarios in which they would be most appropriate.

ADLS offers more granular security than RBAC through the use of access control lists (ACLs) which can be applied at folder of file level.  As per [best practice](https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-best-practices#use-security-groups-versus-individual-users) these should be assigned to AAD groups rather than individual users or service principals. There are two main reasons for this; i.) changing ACLs can take time to propagate if there are 1000s of files, and ii.) there is a limit of 32 ACLs entries per file or folder.

By way of a simple example a data lake may require two sets of permissions. Engineers who run data pipelines and transformations requiring read-write access to a particular set of folders, and analysts who consume [read-only] curated analytics from another. Two AAD groups should be created to represent this division of responsibilities, and the required permissions for each group can be controlled through ACLs. Users should be assigned to the appropriate AAD group, and the group should then be assigned to the ACLs.  Please see [the documentation](https://docs.microsoft.com/en-gb/azure/storage/blobs/data-lake-storage-access-control#access-control-lists-on-files-and-directories) for further details. For automated jobs, a service principal should be used instead of user identity, and these too can be added to the appropriate group. Service principal credentials should be kept extremely secure and referenced only though [secret scopes](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/api/latest/secrets).

For clarity and brevity ADLS in the context of this paper can be considered a v2 storage account with Hierarchical Namespace (HNS) enabled.

#
