# On Prem SQL To Azure API with Logic Apps

<h2>Scenario</h2>
Customer wants to expose a subset of On Premise SQL Database data (multiple databases, specific tables / views) as combined (so related data from different DB’s into single) API’s to the outside world.

<h2>Solution design:</h2> 

![image](https://user-images.githubusercontent.com/23191445/198023098-6f7c11d3-3431-482f-8c42-e5514889b84f.png)

The solution makes use of an On Prem database(s) (1) with SQL Replication (2) to Azure SQL (3). Azure SQL supports the concept of Remote Tables (4), which allow you to query cross databases as if it is one database. Logic Apps (5) is a very easy way to convert SQL table data into JSON over a HTTPS endpoint. API Management (6) exposes the Logic App endpoint as an API and takes care of caching (7).

<h2>System flow</h2>

<h3>1 - On Prem SQL Database</h3>
The On Prem database is mimicked using a SQL 2016 VM, available from the Azure Marketplace: 
    <a href="https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview?view=azuresql">Overview of SQL Server on Azure Windows Virtual Machines - SQL Server on Azure VMs | Microsoft Learn </a><br/>
<h3>2 - SQL Replication </h3>
consists of two parts;<br/>
	- a You create publications of some or all of the data in the SQL Database. There are different types of Replication as can be read here: <a 	href="https://learn.microsoft.com/en-us/sql/relational-databases/replication/types-of-replication?view=sql-server-ver16">Types of Replication - SQL Server | Microsoft Learn</a>
	This sample uses Transactional Replication; The Publisher streams transactions to the Subscribers after they receive an initial snapshot of the published data.
	In this sample we use two Publications from the same database, to sent (related) data to different Azure SQL databases.
	I’m using SQL Server Management Studio (SSMS), which can be downloaded here: <a href="https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms">SQL Server download page</a> <br/> 
	- b The target system can use subscriptions to receive (pull or get pushed) the publicated data. 
	In this sample, the receiving database is an Azure SQL database, which does not have the Agent running, so we use the On Prem SQL Agent to push the data from On Prem to Azure. This is also convenient from a security perspective as the connection is outbound. <br/>
<h3>3 - Azure SQL Databases</h3>
As stated, in this sample there are two target databases, on to get the Persons table and one to get the Address table (and a linking table that holds relation between the two tables).<br/>
<h3>4 - Remote Tables</h3>
the Persons database, we have created two Remote Tables, one for the Address and one for the linking table <br/>
<h3>5 - The Logic App</h3>

![image](https://user-images.githubusercontent.com/23191445/198030105-d4be095c-35d6-48ed-8742-91909b5e788d.png)

The Logic App constist of 3 steps:
1) HTPP POST Trigger that takes a JSON body with one parameter named LastName.
2) SQL Query that SELECTS * FROM the earlier mentioned VIEW WHERE LastName equals the parameter from step 1
3) The Query Result as a JSON response (The Logic App takes care of the formatting) 

<h3>6 - API Management</h3>
I've used the Azure Portal to quickly connect API Management to my backend Logic App

![image](https://user-images.githubusercontent.com/23191445/199545054-b26834a5-0888-4f1e-bc21-1c02f84d7fe5.png)

And used the following Policies to use the Logic App with build-in caching

```
<policies>
    <inbound>
        <base />
        <!-- get lastname from Request.Body. NOTE: use the 'preserveContent: true' directive to stop APIM from flushing it -->
	<set-variable name="LastName" value="@(((JValue)context.Request.Body.As<JObject>(preserveContent: true)["LastName"]).ToString())" />
        <!-- Check if body for this lastname can be retrieved from cache, store result in CachedLastName variable -->
	<cache-lookup-value key="@("ln" + (string)context.Variables["LastName"])" variable-name="CachedLastName" />
        <choose>
	    <!-- if the cache did not return a value, redirect to the Logic App backend to retrieve it from the SQL DB --> 
            <when condition="@(!context.Variables.ContainsKey("CachedLastName"))">
                <set-backend-service id="apim-generated-policy" backend-id="LogicApp_mylogicapp_ce8f0ca1212c45729e98ef56aaf20013" />
            </when>
	    <!-- if it did, return the cached value as Response-->
            <otherwise>
                <return-response>
                    <set-status code="200" reason="Ok" />
                    <set-body>@((string)context.Variables["CachedLastName"])</set-body>
                </return-response>
            </otherwise>
        </choose>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
        <!--Store result 5 minutes in cache    -->
        <cache-store-value key="@("ln" + context.Variables["LastName"])" value="@(context.Response.Body.As<string>(preserveContent: true))" duration="300" />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>

```




