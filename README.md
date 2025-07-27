# azure-serverless-cost-opt
Azure Serverless Architecture - Cost optimization for GET Requests for records fetched via DB and Storage Solution

# Problem Statement
We have a serverless architecture in Azure, where one of our services stores billing records in Azure Cosmos DB. The system is read-heavy, but records older than three months are rarely accessed.

Over the past few years, the database size has significantly grown, leading to increased costs. We need an efficient way to reduce costs while maintaining data availability.

# Current System Constrains
1. Record Size: Each billing record can be as large as 300 KB.

2. Total Records: The database currently holds over 2 million records.

3. Access Latency: When an old record is requested, it should still be served, with a response time in the order of seconds.

# Solution Requirements
Propose a detailed solution to optimize costs while ensuring the following
1. Simplicity & Ease of Implementation – The solution should be straightforward to deploy and maintain.

2. No Data Loss & No Downtime – The transition should be seamless, without losing any records or requiring2 service downtime.

3. No Changes to API Contracts – The existing read/write APIs for billing records must remain unchanged




