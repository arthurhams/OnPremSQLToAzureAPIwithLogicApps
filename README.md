# OnPremSQLToAzureAPIwithLogicApps

Scenario
Customer wants to expose a subset of On Premise SQL Database data (multiple databases, specific tables / views) as combined (so related data from different DB’s into single) API’s to the outside world.
Solution design:
 
The solution makes use of an On Prem database(s) (1) with SQL Replication (2) to Azure SQL (3). Azure SQL supports the concept of Remote Tables (4), which allow you to query cross databases as if it is one database. Logic Apps (5) is a very easy way to convert SQL table data into JSON over a HTTPS endpoint. API Management (6) exposes the Logic App endpoint as an API and takes care of caching (7).

![image](https://user-images.githubusercontent.com/23191445/198023098-6f7c11d3-3431-482f-8c42-e5514889b84f.png)

The solution makes use of an On Prem database(s) (1) with SQL Replication (2) to Azure SQL (3). Azure SQL supports the concept of Remote Tables (4), which allow you to query cross databases as if it is one database. Logic Apps (5) is a very easy way to convert SQL table data into JSON over a HTTPS endpoint. API Management (6) exposes the Logic App endpoint as an API and takes care of caching (7).

1)	The On Prem database is mimicked using a SQL 2016 VM, available from the Azure Marketplace: Overview of SQL Server on Azure Windows Virtual Machines - SQL Server on Azure VMs | Microsoft Learn
2)	SQL Replication consists of two parts;
a.	You create publications of some or all of the data in the SQL Database. There are different types of Replication as can be read here: Types of Replication - SQL Server | Microsoft Learn
This sample uses Transactional Replication; The Publisher streams transactions to the Subscribers after they receive an initial snapshot of the published data.
In this sample we use two Publications from the same database, to sent (related) data to different Azure SQL databases.
I’m using SQL Server Management Studio (SSMS), which can be downloaded here:


b.	The target system can use subscriptions to receive (pull or get pushed) the publicated data. 
In this sample, the receiving database is an Azure SQL database, which does not have the Agent running, so we use the On Prem SQL Agent to push the data from On Prem to Azure. This is also convenient from a security perspective as the connection is outbound. 
3)	 As stated, in this sample there are two target databases, on to get the Persons table and one to get the Address table (and a linking table that holds relation between the two tables).
4)	In the Persons database, we have created two Remote Tables, one for the Address and one for the linking table 
5)	
