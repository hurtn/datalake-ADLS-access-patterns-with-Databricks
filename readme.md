# Securing access to Azure Data Lake Gen 2 from Azure Databricks

**Summary:**

This document attempts to provide customers of Azure Data Lake (ADLS) and Azure Databricks with an introduction of the common approaches in which data can be accessed and secured in ADLS from Databricks.

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

There are a number of ways to configure access to Azure Data Lake Storage gen2 (ADLS) from Azure Databricks (ADB). This document attempts to cover the common patterns, advantages and disadvantages of each, and the scenarios in which they would be most appropriate.

ADLS offers more granular security than RBAC through the use of access control lists (ACLs) which can be applied at folder of file level.  As per [best practice](https://docs.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-best-practices#use-security-groups-versus-individual-users) these should be assigned to AAD groups rather than individual users or service principals. There are two main reasons for this; i.) changing ACLs can take time to propagate if there are 1000s of files, and ii.) there is a limit of 32 ACLs entries per file or folder.

By way of a simple example a data lake may require two sets of permissions. Engineers who run data pipelines and transformations requiring read-write access to a particular set of folders, and analysts who consume [read-only] curated analytics from another. Two AAD groups should be created to represent this division of responsibilities, and the required permissions for each group can be controlled through ACLs. Users should be assigned to the appropriate AAD group, and the group should then be assigned to the ACLs.  Please see [the documentation](https://docs.microsoft.com/en-gb/azure/storage/blobs/data-lake-storage-access-control#access-control-lists-on-files-and-directories) for further details. For automated jobs, a service principal should be used instead of user identity, and these too can be added to the appropriate group. Service principal credentials should be kept extremely secure and referenced only though [secret scopes](https://docs.microsoft.com/en-us/azure/databricks/dev-tools/api/latest/secrets).

For clarity and brevity ADLS in the context of this paper can be considered a v2 storage account with Hierarchical Namespace (HNS) enabled.

#
Pattern 1. Access via Service Principal
=======================================

To provide a group of users access to a particular folder (and it's
contents) in ADLS, the simplest mechanism is to create a [mount point
using a service
principal](https://docs.microsoft.com/en-gb/azure/databricks/data/data-sources/azure/azure-datalake-gen2?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-gb%2Fazure%2Fazure-databricks%2FTOC.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-gb%2Fazure%2Fbread%2Ftoc.json#--mount-an-azure-data-lake-storage-gen2-account-using-a-service-principal-and-oauth-20)
at the desired folder depth. The mount point (/mnt/\<mount\_name\>) is
created once-off per workspace but **is accessible to any user on any
cluster in that workspace**. In order to secure access to different
groups of users with different permissions, one will need more than just
a single one mount point in one workspace. One of the patterns described
below should be followed.

*Note access keys are not an option on ADLS whereas they can be used for
normal blob containers without HNS enabled.*

Below is sample code to authenticate via a SP using OAuth2 and create a
mount point in Scala.

```scala
configs = {"fs.azure.account.auth.type": "OAuth",
           "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
           "fs.azure.account.oauth2.client.id": "enter-your-service-principal-application-id-here",
           "fs.azure.account.oauth2.client.secret": dbutils.secrets.get(scope = "enter-your-key-vault-secret-scope-name-here", key = "enter-the-secret"),
           "fs.azure.account.oauth2.client.endpoint": "https://login.microsoftonline.com/enter-your-tenant-id-here/oauth2/token"}

dbutils.fs.mount(
  source = "abfss://file-system-name@storage-account-name.dfs.core.windows.net/folder-path-here",
  mount_point = "/mnt/mount-name",
```
The creation of the mount point and listing of current mount points in
the workspace can be done via the
[CLI](https://docs.microsoft.com/en-gb/azure/databricks/dev-tools/cli/dbfs-cli?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-gb%2Fazure%2Fazure-databricks%2FTOC.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-gb%2Fazure%2Fbread%2Ftoc.json)

```cli
\>databricks configure --- token

Databricks Host (should begin with ):
[[https://eastus.azuredatabricks.net/?o=\#]{.underline}](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Feastus.azuredatabricks.net%2F%3Fo%3D2614643005307751&data=02%7C01%7CNick.Hurt%40microsoft.com%7C2b8eb8d700c941faea2f08d79cc3869b%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C637150235898192321&sdata=0J%2BFUffNOEIBZWa2VGUqGcX1Z958nuN1r0wRM2xn0HY%3D&reserved=0)\#\#\#\#\#\#\#\#Token:
dapi\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\
\>databricks fs ls dbfs:/mnt

datalake
```
From an architecture perspective these are the basic components where
"dl" is used to represent the mount name.

![](media/image1.png){width="6.268055555555556in"
height="3.4347222222222222in"}

*Note the use of default ACLs otherwise any new folders created will be
inaccessible*

The mount point and ACLs could be at the filesystem (root) level or at
the folder level to grant access at the required filesystem depth.

Instead of mount points, access can also be via direct path --- Azure
Blob Filesystem (ABFS - included in runtime 5.2 and above) as shown in
the code snippet below.

To access data directly using service principal, authorisation code must
be executed in the same session prior to reading/writing the data for
example:

Using a single service principal to authenticate users to a single
location in the lake is unlikely to satisfy most security requirements
-- it is too coarse grained, much like RBAC on Blob containers. It does
not facilitate securing access to multiple groups of users of the lake
who require different sets of permissions. One or more the following
patterns may be followed to achieve the required level of
granularity`

