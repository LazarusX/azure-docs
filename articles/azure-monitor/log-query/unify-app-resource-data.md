---
title: Unify multiple Azure Monitor Application Insights resources  | Microsoft Docs
description: This article provides details on how to use a function in Azure Monitor Logs to query multiple Application Insights resources and visualize that data. 
services: azure-monitor
documentationcenter: ''
author: mgoedtel
manager: carmonm
editor: ''
ms.service: azure-monitor
ms.workload: na
ms.tgt_pltfrm: na
ms.topic: conceptual
ms.date: 01/10/2019
ms.author: magoedte
---

# Unify multiple Azure Monitor Application Insights resources 
This article describes how to query and view all your Application Insights application log data in one place, even when they are in different Azure subscriptions, as a replacement for the deprecation of the Application Insights Connector.  

## Recommended approach to query multiple Application Insights resources 
Listing multiple Application Insights resources in a query can be cumbersome and difficult to maintain. Instead, you can leverage function to separate the query logic from the applications scoping.  

This example demonstrates how you can monitor multiple Application Insights resources and visualize the count of failed requests by application name. Before you begin, run this query in the workspace that is connected to Application Insights resources to get the list of connected applications: 

```
ApplicationInsights
| summarize by ApplicationName
```

Create a function using union operator with the list of applications, then save the query as function with the alias *applicationsScoping*.  

```
union withsource=SourceApp 
app('Contoso-app1').requests,  
app('Contoso-app2').requests, 
app('Contoso-app3').requests, 
app('Contoso-app4').requests, 
app('Contoso-app5').requests 
| parse SourceApp with * "('" applicationName "')" *  
```

>[!NOTE]
>You can modify the listed applications at any time by navigating to Query explorer in the Logs portal and editing the function, or using the `SavedSearch` PowerShell cmdlet. The `withsource= SourceApp` command adds a column to the results that designates the application that sent the log. 
>
>The query uses Application Insights schema, although the query is executed in the workspace since the applicationsScoping function returns the Application Insights data structure. 
>
>The parse operator is optional in this example, it extracts the application name from SourceApp property. 

You are now ready to use applicationsScoping function in the cross-resource query. The function alias returns the union of the requests from all the defined applications. The query then filters for failed requests and visualizes the trends by application. ![Cross-query results example](media/unify-app-resource-data/app-insights-query-results.png)

## Query across Application Insights resources and workspace data 
When you stop the Connector and need to perform queries over a time range that was trimmed by Application Insights data retention (90 days), you need to perform [cross-resource queries](../../azure-monitor/log-query/cross-workspace-query.md) on the workspace and Application Insights resources for an intermediate period. This is until your applications data accumulates per the new Application Insights data retention mentioned above. The query requires some manipulations since the schemas in Application Insights and the workspace are different. See the table later in this section highlighting the schema differences. 

For example, if the connector stopped working on 2018-11-01, when you query logs across Application Insights resources and applications data in the workspace, your query would be constructed like the following example:

```
applicationsScoping //this brings data from Application Insights resources 
| where timestamp between (datetime("2018-11-01") .. now()) 
| where success == 'False' 
| where duration > 1000 
| union ( 
    ApplicationInsights //this is Application Insights data in Log Analytics worspace 
    | where TimeGenerated < (datetime("2018-12-01") 
    | where RequestSuccess == 'False' 
    | where RequestDuration > 1000 
    | extend duration = RequestDuration //align to Application Insights schema 
    | extend timestamp = TimeGenerated //align to Application Insights schema 
    | extend name = RequestName //align to Application Insights schema 
    | extend resultCode = ResponseCode //align to Application Insights schema 
    | project-away RequestDuration , RequestName , ResponseCode , TimeGenerated 
) 
| project timestamp , duration , name , resultCode 
```

## Application Insights and Log Analytics workspace schema differences
The following table shows the schema differences between Log Analytics and Application Insights.  

| Log Analytics workspace properties| Application Insights resource properties|
|------------|------------| 
| AnonUserId | user_id|
| ApplicationId | appId|
| ApplicationName | appName|
| ApplicationTypeVersion | application_Version |
| AvailabilityCount | itemCount |
| AvailabilityDuration | duration |
| AvailabilityMessage | message |
| AvailabilityRunLocation | location |
| AvailabilityTestId | id |
| AvailabilityTestName | name |
| AvailabilityTimestamp | timestamp |
| Browser | client_browser |
| City | client_city |
| ClientIP | client_IP |
| Computer | cloud_RoleInstance | 
| Country | client_CountryOrRegion | 
| CustomEventCount | itemCount | 
| CustomEventDimensions | customDimensions |
| CustomEventName | name | 
| DeviceModel | client_Model | 
| DeviceType | client_Type | 
| ExceptionCount | itemCount | 
| ExceptionHandledAt | handledAt |
| ExceptionMessage | message | 
| ExceptionType | type |
| OperationID | operation_id |
| OperationName | opeation_Name | 
| OS | client_OS | 
| PageViewCount | itemCount |
| PageViewDuration | duration | 
| PageViewName | name | 
| ParentOpeartionID | operation_Id | 
| RequestCount | itemCount | 
| RequestDuration | duration | 
| RequestID | id | 
| RequestName | name | 
| RequestSuccess | success | 
| ResponseCode | resultCode | 
| Role | cloud_RoleName |
| RoleInstance | cloud_RoleInstance |
| SessionId | session_Id | 
| SourceSystem | operation_SyntheticSource |
| TelemetryTYpe | type |
| URL | _url |
| UserAccountId | user_AccountId |

## Next steps

Use [Log Search](../../azure-monitor/log-query/log-query-overview.md) to view detailed information for your Application Insights apps.