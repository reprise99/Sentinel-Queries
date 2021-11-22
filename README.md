# KQL for Microsoft Sentinel

Some tips, tricks and examples for using KQL for Microsoft Sentinel.

1. [Introduction](#introduction)
2. [The Anatomy of a KQL Query](#the-anatomy-of-a-kql-query)
3. [The Basics](#the-basics)
    1. [Time Basics](#time-basics)
    2. [Where Basics](#where-basics)
    3. [Project Basics](#project-basics)
    4. [Summarize Basics](#summarize-basics)

## Introduction

Kusto Query Language is the language used across Azure Monitor, Azure Data Explorer and Azure Log Analytics (what Microsoft Sentinel uses under the hood). I have always found this visualization regarding KQL useful -

![KQL visualized](https://github.com/reprise99/Sentinel-Queries/blob/main/Diagrams/kql-pipe.png?raw=true)

We want to use KQL to create accurate and efficient queries to find threats, detections, patterns and anomalies from within our larger data set.

## The Anatomy of a KQL Query

Take the below query as an example

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| where AppDisplayName == "Microsoft Teams"
| project TimeGenerated, Location, IPAddress, UserAgent
```

When we run a query like this the first line tells Microsoft Sentinel which table to look for data in, so in this case we want to search the SigninLogs table, which is where Azure AD sign in data is sent to. You can see a list of tables [here](https://docs.microsoft.com/en-us/azure/sentinel/data-source-schema-reference).

Microsoft Sentinel will then run through your query sequentially, so it will run each line one by one until it hits the end, or you have an error. So to breakdown our query line by line.

```kql
SigninLogs
```

So first we have chosen our SigninLogs table.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
```

Next we tell Sentinel to look back at the last 14 days worth of data in this table.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
```

Next we ask Sentinel to only find logs where UserPrincipalName is equal to "reprise_99@testdomain.com"

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
```

Then we look for only logs where the ResultType == 0, which is a successful logon to Azure AD.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| where AppDisplayName == "Microsoft Teams"
```

Next we look for only signins to Microsoft Teams.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| where AppDisplayName == "Microsoft Teams"
| project TimeGenerated, Location, IPAddress, UserAgent
```

Our last line uses the project operator, to return only 4 fields from our logs, so we will only see the TimeGenerated, Location, IPAddress and UserAgent returned from our SigninLogs data.

That is how you build queries, now the basics.

## The Basics

### Time Basics

Microsoft Sentinel and KQL are highly optimized for time filters, so if you know the time period of data you want to search, you should filter the time range straight away. Retrieving the last 14 days of logs, then searching for a username like the below query -

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
```

Is much more efficient than searching first for a username and then searching for the time period like this -

```kql
SigninLogs
| where UserPrincipalName == "reprise_99@testdomain.com"
| where TimeGenerated > ago(14d)
```

KQL has many options for querying particular time periods.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
```

As per the first example, this will search for the last 14 days.

```kql
SigninLogs
| where TimeGenerated > ago(14h)
```

You can also do hours.

```kql
SigninLogs
| where TimeGenerated > ago(14m)
```

And minutes.

KQL also supports querying between time ranges -

```kql
SigninLogs
| where TimeGenerated between (ago(14d) .. ago(7d))
```

This will find SigninLogs data between 14 days and 7 days ago.

```kql
SigninLogs
| where TimeGenerated between (ago(14h) .. ago(7h))
```

Between 14 hours and 7 hours ago.

```kql
SigninLogs
| where TimeGenerated between (ago(14m) .. ago(7m))
```

And between 14 minutes and 7 minutes ago.

### Where Basics

Where is an operator you will use in basically every query you write. This is how you tell Microsoft Sentinel to hunt for specific data. Syntax is very important with the where operator. If we use our same example.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
```

This will search our SigninLogs table, over the last 14 days, for exact matches where our UserPrincipalName equals reprise_99@testdomain.com. In KQL == is case sensitive, so if you search for reprise_99@TESTdomain.com and the username is actually reprise_99@testdomain.com, you won't get any results. The non case sensitive equivalent is =~

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName =~ "reprise_99@TESTdomain.com"
```

This will find any matches for reprise_99@testdomain.com regardless of case sensitivity.

Instead of equals, we can also use contains.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName contains "reprise_99"
```

This will find any log entries where the UserPrincipalName contains reprise_99, if you had reprise_99@testdomain.com and reprise_99@anotherdomain.com data, it would find both. The contains operator is not case sensitive, but you can use contains_cs to make it case sensitive.

You can use either startswith or endswith if you are searching for particular patterns.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName startswith "reprise_99"

SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName endswith "testdomain.com"
```

Both startswith and endswith are not case sensitive, but you can use the startswith_cs or endswith_cs to make them case sensitive.

If you are searching for full words (greater than four characters), in KQL you can use the has operator. Using 'has' is more efficient than 'contains' as the data is indexed for you.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where AppDisplayName has "Teams"
```

This will find any SigninLogs where the application display name has the word Teams in it, that could include "Microsoft Teams" and "Microsoft Teams Web Client", both satisfy the query.

If you are searching for multiple words you can use has_any or has_all.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where AppDisplayName has_any ("Teams","Outlook")
```

This will return results where the application display name contains either "Teams" or "Outlook"

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where AppDisplayName has_all ("Teams","Outlook")
```

This will return results where the application display name has "Teams" and "Outlook".

If you don't know which fields to search in, you can also use wildcards, it is inefficient but may get you on the right track to find what you want.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where * contains "reprise_99"
```

This will search the SigninLogs table for any field that contains reprise_99.

A number of these options also support using ! to reverse the query and find results where it is not true.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName != "reprise_99@testdomain.com"
```

This query would find all SigninLogs where the UserPrincipalName does not equal reprise_99@testdomain.com

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName !contains "reprise_99"
```

This query would find all SigninLogs where the UserPrincipalName does not contain reprise_99

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where AppDisplayName !has "Teams"
```

This query would find SigninLogs where the application display name does not contain "Teams".

### Project Basics

Project allows us to select which columns are returned in our query and in which order.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| where AppDisplayName == "Microsoft Teams"
| project TimeGenerated, Location, IPAddress, UserAgent
```

This query searches for SigninLogs data from the last 14 days, where the UserPrincipalname equals reprise_99@testdomain.com, where the ResultType is 0, where the application display name equals "Microsoft Teams", then for each match on that query it returns the TimeGenerated, Location, IPAddress and the UserAgent.

We can rename colums as part of the same function.

```kql
| project LogTime=TimeGenerated, SigninLocation=Location, IP=IPAddress, Agent=UserAgent
```

This returns the same data, but renames the columns to LogTime, SigninLocation, IP and Agent.

We can even manipulate the output inline with the project operator.

```kql
| project LocalTime=TimeGenerated+5h, Location, IPAddress, UserAgent
```

This returns the same data, but changes the TimeGenerated name to LocalTime and converts to a +5h time zone if you work in that time zone.

project-away is the opposite of project and will remove columns from your query.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| project-away UserAgent
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| where AppDisplayName == "Microsoft Teams"
```

In this query we remove UserAgent. Remember, if you remove a column you then can't access it later in your query.

### Summarize Basics

Summarize produces a table that aggregates the content of your query. Summarize has a number of underlying aggregation functions. If we again take our example query, we can manipulate the results in various ways using summarize.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| summarize count() by AppDisplayName
```

This query will look up the SigninLogs table for any events in the last 14 days, for any matches for reprise_99@testdomain.com, where the result is a success (ResultType == 0) and then summarize those events by the application display name.

You can optionally name the result column.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| summarize AppCount=count() by AppDisplayName
```

This returns the same data but updates the name of the returned column to AppCount.

Instead of a total count, you can summarize a distinct count.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| summarize DistinctAppCount=dcount(AppDisplayName) by AppDisplayName
```

This will return a single record for each distinct application reprise_99@testdomain.com signed into.

You can use the arg_max and arg_min functions to return either the newest or oldest record that matches your query.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| summarize arg_max(TimeGenerated, *) by UserPrincipalName
```

This query looks for all signin logs over the last 14 days, that have reprise_99@testdomain.com as the UserPrincipalname, that are successful and then returns the latest record.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| summarize arg_min(TimeGenerated, *) by UserPrincipalName
```

This is the same but returns the oldest record.

You can use countif to provide logic to your summations.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| summarize TeamsLogons=countif(AppDisplayName has "Teams"), SharePointLogons=countif(AppDisplayName has "SharePoint")
```

This summarizes the data into two new columns, TeamsLogons where the application display name has "Teams" and SharePointLogons where the application display name has "SharePoint"

You can further manipulate your data by telling KQL to place your data into time 'bins'.

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| summarize AppCount=count() by AppDisplayName, bin(TimeGenerated, 1d)
```

This returns the same data as our first summarize example and then groups that data into 1d bins.

You can combine these functions together where useful

```kql
SigninLogs
| where TimeGenerated > ago(14d)
| where UserPrincipalName == "reprise_99@testdomain.com"
| where ResultType == "0"
| summarize TeamsLogons=countif(AppDisplayName has "Teams"), SharePointLogons=countif(AppDisplayName has "SharePoint") by bin(TimeGenerated, 1d)
```

This is a combination of our countif and bin functions, where we summarize based on our application display name and also place the results into 1d bins.
