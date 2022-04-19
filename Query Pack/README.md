# Query Packs

Query packs are an Azure Resource Manager object that act as a container to store queries. This provides a way to save KQL queries and share them across multiple workspaces, or to the community.

This feature is currently in preview but the guidance can be found [here](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/query-packs)

You can deploy the queries from this repo to your Azure environment by deploying the following template.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Freprise99%2FSentinel-Queries%2Fmain%2FQuery%2520Pack%2Fazuredeploy.json)

You will need to choose an Azure subscription, Azure resource group and Azure region to deploy to. You can rename the query pack if you want to.

![Deploy to Azure](https://github.com/reprise99/sentinel-queries/blob/main/Diagrams/deploytoazure.png?raw=true)

Once deployed, you can then find all the queries in your Microsoft Sentinel or Log Analytics workspace.

## Finding your queries

1. Select your query pack by selecting 'Queries', then clicking the 3 dots here.

![Query Pack 1](https://github.com/reprise99/sentinel-queries/blob/main/Diagrams/querypack1.png?raw=true)

2. You can select just one, or multiples.

![Query Pack 2](https://github.com/reprise99/sentinel-queries/blob/main/Diagrams/querypack2.png?raw=true)

3. The queries have all been labelled by which technology they belong to, you can sort by labels here.

![Query Pack 3](https://github.com/reprise99/sentinel-queries/blob/main/Diagrams/querypack3.png?raw=true)

4. You will see a list of all the queries in the query pack.

They are organized into technology, and then into 4 sub categories

    Detect - for detection based queries.
    Find - for finding results you may be interested in but are not necessarily a threat.
    Summarize - for queries that summarize results.
    Visualize - for queries that display a visualization of results.

![Query Pack 4](https://github.com/reprise99/sentinel-queries/blob/main/Diagrams/querypack4.png?raw=true)

5. Each query should have a description for what it looks for.

![Query Pack 5](https://github.com/reprise99/sentinel-queries/blob/main/Diagrams/querypack5.png?raw=true)

6. Select run to run the query. You can also edit the queries to make them more relevant to your environment then re-save it to your deployed Query Pack.