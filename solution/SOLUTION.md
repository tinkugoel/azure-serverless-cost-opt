# Archival of Data Older than 3 months from Azure Cosmos DB
We will archive the older billing records from Cosmos DB to **Azure BLOB Storage**. Reasons for choosing this option:-
1. Blob Storage can be used to store documents, files of any size so no size constrains in future if record size increases
2. Blob Storage capacity is dynamic, so we do not need to estimate and provision storage capacity.
3. Blob Storage provisioned under **Standard Storage accounts are the most cost-effective solution** and do not attract high costs.
```
# Create a storage account with Standard Sku
az storage account create \
    --name <yourstorageaccount> \
    --resource-group <rg> \
    --location <location> \
    --sku Standard_LRS \
    --access-tier Cool \
    --kind StorageV2
```
4. We can provision Blob Storage with **COLD Tier** for low read/retrieval operation in case our data is accessed rarely whereas in Archive Tier the retrieval cost is much higher.
```
# Create a blob container for archived records with Cold Tier
az storage container create \
    --name <container-name> \
    --account-name <storage-account-name> \
    --public-access off
```
5. Cold Tier provides records retrieval latency in order of seconds whereas it is hours in case of Archive Tier.

# Migration of records

**For initial migration of ~2M records, we need to do batch processing by managing throttling for which Azure Data Factory is best solution.**

We will use **Azure Data Factory** instead of **Azure Functions** as implementing a parallel, incremental archival solution using Azure Data Factory pipeline is easy to deploy and manage.

Using these pipelines, we can also monitor the progres, handle batch processing to minimize load on Azure Cosmos DB to smooth up the operations. Instead in case of Azure Functions, we have the possibility of timeouts, complex logic implementation, slightly higher cost than Azure Data Factory pipelines.

Also, Azure Functions are more error prone if logic is not handled efficiently, so need to implement advanced retry mechanisms which may result in longer execution times and ultimately exceeding cost.

**We can follow the below steps to implement migration of records with pipelines in Azure Data Factory using GUI interface** 

1. Choose Copy Operation
2. Select Source as Azure Cosmos DB container
3. Create Cosmos DB Linked Service (with connection details/auth)
4. Apply filter for source records for timestamp >= 3 months
5. Select Destination as from Azure Blob Storage container
6. Create Blob Storage Linked Service (with connection string/key)
7. Provide required properties for destination.

Post this our copy operation is complete. Now we need to configure for the **Post-Copy-Deletion part**.

For this we will orchestrated Bulk Delete via Stored Procedure in an ADF Pipeline. ADF can trigger deletion indirectly—by invoking a Cosmos DB stored procedure that performs batch deletes:

1. Create a Cosmos DB Stored Procedure
*Pseudocode for same*
```
function bulkDeleteOlderThan(cutoffDate) {
    var collection = getContext().getCollection();
    var isAccepted = collection.queryDocuments(
        collection.getSelfLink(),
        "SELECT c._self FROM c WHERE c.timestamp < '" + cutoffDate + "'",
        function (err, docs) {
            // (iterate docs array, delete each, return count and continuation)
        }
    );
}
```
2. Add Stored Procedure to Cosmos DB Container
3. Before deletion we will validate if records are stored in blob storage or not. For this Use a Lookup/If Activity to compare expected vs. actual count, and confirm blob existence. If validation fails, stop pipeline and send alert.
4. Build the ADF Pipeline with below steps for invoking deletion activity:-

   a. Pre-step: Calculate the data to pass it to the Stored Procedure as parameter
   
   b. Web Activity: Use ADF Web Activity to invoke the stored procedure, passing the cutoff date as a parameter only on validation success.

# For records retrieval from Azure Cosmos DB and Azure BLOB Storage(in case of older >= 3 months)

API Contracts with client remains same and no changes needs to be informed to clients/end users, however, we need to implement a fallback logic in our backend for that GET API call.

As our old records are moved to Azure Blob Storage and no longer reside in Cosmos DB, so we need to include this handling in backend logic. 

**Sample Pseudocode:-** which will check if the record exists in Cosmos DB first, if not found, then fallback to Blob Storage
```
def get_billing_record(record_id):
    # Try Cosmos DB First
    cosmos_record = cosmos_db.get(record_id)
    if cosmos_record:
        return cosmos_record
    else:
        # Fallback - Try to fetch from Blob Storage
        blob_record = blob_storage.get_blob(record_id)
        if blob_record:
            return blob_record
        else:
            # notify about error that record is not found by raising an exception or handling it with frontend interface integration 
            return None
        
```
