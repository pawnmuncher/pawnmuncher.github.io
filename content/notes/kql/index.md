---
title: KQL Notes
menu:
  notes:
    name: KQL
    identifier: notes-KQL
    weight: 100
---
https://detective.kusto.io/inbox

{{< note title="Season 2 - Case 2 - Catch the Phishermen!" >}}
```SQL
// Training

//Get the schema, know what we are working with
PhoneCalls
| getschema

//see how big a data set
PhoneCalls
| count 

//see some example data
PhoneCalls
| take 100

//parse "out" the json data using dotNotation so we can use it to filter
PhoneCalls 
| where EventType == 'Connect' 
| extend Origin=tostring(Properties.Origin), Destination=tostring(Properties.Destination), Hidden=tobool(Properties.IsHidden) 
| take 10

//Test 1 using parsed json as filters
PhoneCalls 
| where EventType == 'Connect' 
| where tobool(Properties.IsHidden) == false or Properties !has 'IsHidden'
| summarize Count=count() by Phone=tostring(Properties.Origin) 
| top 1 by Count

//Perform some statistics
PhoneCalls 
| where EventType =='Connect' 
| summarize calls=count() by bin(Timestamp, 1h) 
| summarize avg(calls), percentile(calls, 50)

//Use join 
PhoneCalls 
| where EventType == 'Connect'
| where Properties.Destination != '0' 
| join kind=inner
    (PhoneCalls
    | where EventType == 'Disconnect'
    | extend DisconnectProperties = Properties) 
    on CallConnectionId 
| where DisconnectProperties.DisconnectedBy == 'Destination'
| count

//Solution
PhoneCalls 
| where EventType == 'Connect'
| extend Origin  = tostring(Properties.Origin)
| extend Destination = tostring(Properties.Destination)  
| where Properties has 'IsHidden'
| join kind=inner
    (PhoneCalls
    | where EventType == 'Disconnect'
    | extend DisconnectedBy = tostring(Properties.DisconnectedBy)
    | where DisconnectedBy == 'Destination')
    on CallConnectionId 
| summarize DistinctDestinations = dcount(Destination) by Origin
| top 1 by DistinctDestinations
```
{{< /note >}}

{{< note title="Season 2 - Case 1 - To bill or not to bill?" >}}
```sql
//Case 1 training
//can run sql directly
SELECT SUM(Consumed * Cost) AS TotalCost 
FROM Costs 
JOIN Consumption ON Costs.MeterType = Consumption.MeterType 

//can use EXPLAIN to convert to KQL
EXPLAIN
SELECT SUM(Consumed * Cost) AS TotalCost 
FROM Costs 
JOIN Consumption ON Costs.MeterType = Consumption.MeterType 

//from EXPLAIN
Costs
| join kind=inner 
(Consumption
| project-rename ['Consumption.MeterType']=MeterType) on (['$left'].MeterType == ['$right'].['Consumption.MeterType'])
| summarize TotalCost=sum(__sql_multiply(Consumed, Cost))
| project TotalCost

//cleaned up a bit USES LOOKUP
Consumption  
 | summarize TotalConsumed = sum(Consumed) by MeterType  
 | lookup Costs on MeterType  
 | extend TotalCost = TotalConsumed*Cost  
 | summarize sum(TotalCost) 

// who used the most water
Consumption
| where MeterType == "Water"
|summarize WaterWaster=sum(Consumed) by HouseholdId
|top 10 by WaterWaster

//who used the most electricity
Consumption
| where MeterType == "Electricity"
| summarize ElectricityWaster=sum(Consumed) by HouseholdId
| top 10 by ElectricityWaster

//Water consumption by date Timestamp
Consumption
| where MeterType == "Water"
| summarize WaterWaster=sum(Consumed) by bin(Timestamp, 1d)
| render timechart 

//Electricity by date
Consumption
| where MeterType == "Electricity"
| summarize Elec_Waster=sum(Consumed) by bin(Timestamp, 1d)
| render timechart 

//Elec stats like min max count and avg
Consumption
| where MeterType =="Electricity"
| summarize Total_Count=count(), Lowest_Value=min(Consumed), Highest_Value=max(Consumed), Average_Value=avg(Consumed)

//Case 1 - Solved
Consumption
| summarize hint.strategy=shuffle arg_max(Consumed, *) by HouseholdId, MeterType, Timestamp
| join kind=inner (
    Costs
    | project MeterType, Cost
) on MeterType
| extend BillAmount = Consumed * Cost
| summarize TotalBillsAmount = sum(BillAmount)

//using lookup function
Consumption
| summarize hint.strategy=shuffle arg_max(Consumed, *) by HouseholdId, MeterType, Timestamp
| lookup Costs on MeterType
| summarize sum(Consumed * Cost)

//Even cleaner using the DISTINCT function
Consumption
| distinct Consumed, HouseholdId, MeterType, Timestamp
| lookup Costs on MeterType
| summarize round(sum(Consumed * Cost),2)

```

{{< /note >}}

{{< note title="Kusto Detective KQL Challenge Information" >}}

```MD
## Welcome to the Kusto Detective Agency - Season 2!
Here you can find my solutions for Season 2.

Feel free to reach out if you have any questions.
```
{{< /note >}}

{{< note title="Season 2 - Case 0 - Onboarding" >}}

```SQL
//Case 0 training
DetectiveCases
| where EventType == 'CaseOpened'
| extend Bounty = toreal(Properties.Bounty)
| count

DetectiveCases
| where EventType == 'CaseSolved'
| summarize CasesSolved=count() by DetectiveId
//| count
| top 10 by CasesSolved

DetectiveCases
| where EventType == 'CaseOpened'
| project CaseOpened = Timestamp, CaseId
| join kind=inner 
    (
    DetectiveCases
    | where EventType == 'CaseAssigned'
    | summarize FirstAssigned=min(Timestamp) by CaseId)
    on CaseId
| summarize Average=avg(FirstAssigned - CaseOpened)

DetectiveCases
| where EventType == 'CaseOpened'
| project CaseOpened = Timestamp, CaseId
| join kind=inner 
    (
    DetectiveCases
    | where EventType == 'CaseSolved'
    | summarize CaseSolved=min(Timestamp) by CaseId)
    on CaseId
| summarize Average=avg(CaseSolved - CaseOpened)

DetectiveCases
| where EventType == 'CaseOpened'
| project CaseOpened=Timestamp, CaseId
| join kind=inner (
    DetectiveCases
    | where EventType == 'CaseSolved'
    | summarize CaseSolved=min(Timestamp) by CaseId
    ) on CaseId
| summarize avg(CaseSolved - CaseOpened)

//Case 0 Solved
DetectiveCases
| where Timestamp >= datetime(2022-01-01) and Timestamp < datetime(2023-01-01)
| where EventType == "CaseOpened" and isnotnull(Properties.Bounty) and isnotnull(CaseId)
| project CaseId, Bounty = todouble(parse_json(Properties).Bounty)
| join kind=inner (
    DetectiveCases
    | where EventType == "CaseAssigned" and isnotnull(DetectiveId) and isnotnull(CaseId)
    | project CaseId, DetectiveId
) on CaseId
| summarize TotalEarnings = sum(Bounty) by DetectiveId
| top 10 by TotalEarnings desc

```

{{< /note >}}