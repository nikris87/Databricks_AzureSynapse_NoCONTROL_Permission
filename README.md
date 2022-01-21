# Databricks Azure Synapse connctor No CONTROL Permission required
 This article talks about how to read and write data from Synapse SQL Pool with minimum permission granted using Databricks connector .

## CONTEXT:

Databricks connector uses Azure Synapse SQL pool native technology - Polybase or Copy Command to read\write data from Synapse dedicated pool. Polybase or Copy leverages the scale-out architecture of Synapse SQL Dedicated pool for high throughput data ingestion\offloading. 

Polybase can be used for both read and write (Ingestion) scenario, Copy command can used only for Ingestion (Write) scenario to dedicated SQL pool.

For more details refer Azure Synapse SQL pool and Databricks connector integration doc : https://docs.databricks.com/data/data-sources/azure/synapse-analytics.html#azure-synapse-analytics

## Problem statement:

Azure databricks users require CONTROL Permission in Synapse dedicated SQL Pool if connector using polybase.

CONTROL permission is elevated permission just like DB owner, granting this level of permission is not acceptable for most customers.

### 1.	Write scenario from Databricks to Synapse Dedicated pool:

For write scenario by default databricks write connector uses Synapse Dedicated pool’s Copy Command statement. 

The COPY statement offers a more convenient way of loading data into Azure Synapse SQL pool without the need to create an external table, and requires fewer permissions to load data, and improves the performance of data ingestion into Azure Synapse.

For write scenario you will not encounter any CONTROL permission challenges as it utilizes the Copy command.

Here is the reference: https://docs.databricks.com/data/data-sources/azure/synapse-analytics.html#write-semantics

You need to grant [Copy command minimal permission](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/quickstart-bulk-load-copy-tsql?context=/azure/synapse-analytics/context/context#set-up-the-required-permissions) for user in Synapse SQL pool.


### 2.	Read scenario from Synapse Dedicated SQL pool to Azure databricks [Data frames]:
For Read scenario only option for databricks connector is to leverage Polybase. 
Polybase requires the following objects to be created prior to each load. Databrick spark.read connector will execute below commands against Dedicated SQL pool using user authentication for each read operation.

1.	[MASTER KEY](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-master-key-transact-sql?view=sql-server-2017) – Requires database CONTROL permission
2.	[DATABASE SCOPED CREDENTIAL](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-database-scoped-credential-transact-sql?view=azure-sqldw-latest) – Requires database CONTROL permission
3.	[EXTERNAL DATA SOURCE](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-external-data-source-transact-sql?view=azure-sqldw-latest&tabs=dedicated) – Requires database CONTROL permission
4.	[EXTERNAL FILE FORMAT](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-external-file-format-transact-sql?view=azure-sqldw-latest&tabs=delimited) - Requires ALTER ANY EXTERNAL FILE FORMAT
5.	[EXTERNAL TABLE](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-external-table-transact-sql?view=azure-sqldw-latest&tabs=dedicated) – Requires: CREATE TABLE, ALTER on the SCHEMA, ALTER ANY EXTERNAL DATA SOURCE, and ALTER ANY EXTERNAL FILE FORMAT

#### Scenario 1: Service principle auth from Databricks:
For this scenario , you can create these 3(step 1-3) objects in advance by a user who has CONTROL permission on the database.

Synapse dedicated pool and Databricks user will use the pre-created External data source in Databricks Connector using [ExternalDataSource](https://docs.databricks.com/data/data-sources/azure/synapse-analytics.html#required-azure-synapse-permissions-for-polybase-with-the-external-data-source-option) parameter option.

Required permission for user at Dedicated sql pool level- CREATE TABLE, ALTER ANY SCHEMA, ALTER ANY EXTERNAL DATA SOURCE, and ALTER ANY EXTERNAL FILE FORMAT.

#### Step 1 : 
[Create Service Principle](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal)

#### Step 2: 
Assign [storage permission](https://docs.microsoft.com/en-us/azure/storage/blobs/assign-azure-role-data-access?tabs=portal) to Service principle to hold intermediate result during polybase. 

Grant Workspace\logical server access to storage if you are planning to use MSI auth to storage.

#### Step 3: Add Service principle user to dedicated SQL Pool and grant below required permissions:

CREATE USER [Service Principle Name] FROM EXTERNAL PROVIDER;

EXEC sp_addrolemember 'db_datareader', 'Service Principle Name';

Grant CREATE TABLE TO [Service Principle Name]

Grant ALTER ANY SCHEMA TO [Service Principle Name]

Grant ALTER ANY EXTERNAL DATA SOURCE TO [Service Principle Name]

Grant ALTER ANY EXTERNAL FILE FORMAT TO [Service Principle Name]

#### Note :If you prefer to grant only schema level access for user follow [this](https://techcommunity.microsoft.com/t5/azure-synapse-analytics-blog/how-to-create-a-polybase-user-with-only-schema-level-access/ba-p/839878) doc.

#### Step 4: 
User with CONTROL permission [create External data source](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-external-data-source-transact-sql?view=azure-sqldw-latest&preserve-view=true&tabs=dedicated#b-create-external-data-source-to-reference-azure-data-lake-store-gen-1-or-2-using-a-service-principal) using with SP\MSI\AAD auth method to storage .

Make a note of external data source name, that required for Databricks Notebook in later stage.

Ex : using MSI auth 
```
CREATE DATABASE SCOPED CREDENTIAL msi_cred WITH IDENTITY = 'Managed Service Identity';

CREATE EXTERNAL DATA SOURCE [DS_imported_MSI] 

	WITH (
  
		LOCATION = 'abfss://<Container>@<Storage name>.dfs.core.windows.net', 
    
		TYPE     = HADOOP ,
    
		CREDENTIAL= msi_cred
    
	)
  ```

#### Step 5 : Use above created data source in databricks read connector:

 ```
df2 = spark.read \
  .format("com.databricks.spark.sqldw") \
  .option("url","jdbc:sqlserver://<Workspace>.sql.azuresynapse.net:1433;  database=<DBname>;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30") \
  .option("tempdir", f"abfss://<Container>@<Storage Name>.dfs.core.windows.net/temp")\
  .option("enableServicePrincipalAuth", "true")\
  .option("externalDataSource", "DS_imported_MSI")\
  .option("dbTable", "dbo.user_details") \
  .load()
 ```



#### Scenario 2: AAD authentication from Databricks notebook and AAD passthrough enabled in cluster.

If you are using  Azure databricks AAD passthrough, as explained in one of our [Synapse blog](https://techcommunity.microsoft.com/t5/azure-synapse-analytics-blog/how-to-use-polybase-by-authenticating-via-aad-pass-through/ba-p/862260) ,CONTROL permission is not required in Synapse SQL pool for AAD passthrough .

Databricks AAD user authentication will pass to the Synapse dedicated pool and storage, this will work without need of a CONTROL permission.

Note: Required permission for user at Dedicated sql pool - CREATE TABLE, ALTER ANY SCHEMA, ALTER ANY EXTERNAL DATA SOURCE, and ALTER ANY EXTERNAL FILE FORMAT.

If you prefer to grant only schema level access for user follow [this](https://techcommunity.microsoft.com/t5/azure-synapse-analytics-blog/how-to-create-a-polybase-user-with-only-schema-level-access/ba-p/839878) doc.

Databricks code Looks something like below for AAD auth:
 ```
df1 = spark.read \
  .format("com.databricks.spark.sqldw") \      .option("url","jdbc:sqlserver://<Workspace.sql.azuresynapse.net:1433;database=<DBname>;encrypt=true;trustServerCertificate=true;hostNameInCertificate=*.database.windows.net;loginTimeout=30") \
  .option("tempdir", f"abfss://<Container>@<StorageName>.dfs.core.windows.net/temp")\
  .option("dbTable", "dbo.DimDate") \
  .load()
   ```

##### And you are good to go!!!




