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
| extend Origin=tostring(Properties.Origin), Destination=tostring(Properties.Destination)
| where Properties has 'IsHidden'
| join kind=inner
    (PhoneCalls
    | where EventType == 'Disconnect'
    | extend DisconnectedBy = tostring(Properties.DisconnectedBy)
    | where DisconnectedBy == 'Destination')
    on CallConnectionId 
| summarize PerpNumber = dcount(Destination) by Origin
| top 1 by PerpNumber

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

{{< note title="Season 2 - Case 3 - Return Stolen Cars!" >}}

```SQL
// Records count
CarsTraffic
| count 

// See some examples
CarsTraffic
| take 100 

// Records count
StolenCars
| count 

//See some examples
StolenCars
| take 100 

// find all rows that have one of a list of VIN numbers
CarsTraffic 
| where VIN in ('FD655964S', 'JO132865F', 'AD701526K') 
| count

//the stolen car with the most sightings
CarsTraffic 
| where VIN in (StolenCars) 
| summarize Sightings=count() by VIN 
| top 1 by Sightings

//which were the first and last cars to be spotted at a certain intersection
CarsTraffic 
| where Street == 180 and Ave == 121 
| summarize First=arg_min(Timestamp, VIN), Last=arg_max(Timestamp, VIN)

//the street and avenue was stolen car with VIN number IR177866Y first sighted
CarsTraffic 
| where VIN == 'IR177866Y' 
| summarize First=arg_min(Timestamp, Street, Ave)

//find all cars that were on the same street at a certain time in both the morning and evening of June 14
CarsTraffic
| where Timestamp between (datetime(2023-06-14 08:00) .. 1h)
| join kind = inner (
    CarsTraffic
    | where Timestamp between (datetime(2023-06-14 20:00) .. 1h)
    ) on VIN, Street, Ave
| summarize dcount(VIN)

//cars have been at Street 228, Ave 145 but have never been sighted at location Street 121, Ave 180
CarsTraffic 
| where Street == 228 and Ave == 145 
| join kind=leftanti (
    CarsTraffic | where Street == 121 and Ave == 180 | distinct VIN) on VIN 
| distinct VIN 
| count

//create the joined data to find the stash house for the stolen cars
//last seen location for stolen cars
let VinsbyLocation =
    StolenCars
    | join kind=inner (CarsTraffic) on $left.VIN == $right.VIN
    | summarize arg_max(Timestamp, Ave, Street) by VIN
    | extend TimeKey = bin(Timestamp, 15m)
    | project TimeKey, Ave, Street;
//all VIN at the same location within 15 time window
let VinsbyTime = VinsbyLocation
    | join kind=inner ( CarsTraffic | extend TimeKey = bin(Timestamp, 15m)) on TimeKey, Ave, Street
    | summarize by VIN;
//last sighting for each vin and most visited.  The top two are where the plates were swapped
//the next is the stash spot
VinsbyTime
| join kind=inner ( CarsTraffic | summarize arg_max(Timestamp, Ave, Street) by VIN ) on VIN
| summarize count() by Ave, Street
| order by count_ desc
```
{{< /note >}}

{{< note title="Season 2 - Case 4 - Triple trouble!" >}}

```SQL
// Triple trouble
NetworkMetrics  
| getschema 

NetworkMetrics
//| count
| take 10

IpInfo
| getschema 

IpInfo
//| count
| take 10

//bin function to create buckets of time
NetworkMetrics
| summarize count() by bin(Timestamp, 1d)
| render timechart

// You can use time-chart to look on the data
NetworkMetrics
| summarize avg(BytesSent) by bin(Timestamp, 1d)
| render timechart

// .. or you can use query to calculate it
NetworkMetrics
| summarize avg(BytesSent) by bin(Timestamp, 1d)
| top 1 by avg_BytesSent asc

NetworkMetrics
| take 10
| evaluate ipv4_lookup(IpInfo, ClientIP, IpCidr)

//Which company most frequently contacted IP address 178.248.55.129
NetworkMetrics
| where TargetIP == '178.248.55.129'
| evaluate ipv4_lookup(IpInfo, ClientIP, IpCidr)
| summarize Count=count() by Info
| order by Count

//The following query creates two series (as two columns): one with the number of records per day and the other with the average amount of data received per day.
NetworkMetrics
| make-series count(), avg(BytesReceived) on Timestamp step 1d
| render timechart with (ysplit=panels)


//I suspect that someone has hacked into the Digitown municipality system and stolen these documents.
// Our system is a known data hub and hosts various information about the town itself, real-time monitoring 
// systems of the city, tax payments, etc. 
// It serves as a real-time data provider to many organizations around the world, so it receives a lot of traffic.
// Unfortunately, I dont have much data to give you. 
// All I have is a 30day traffic statistics report captured by the Digitown municipality system network routers

let sus = 
NetworkMetrics
| summarize dcount(TargetIP) by ClientIP
| where dcount_TargetIP == 1 
| distinct ClientIP
| evaluate ipv4_lookup(IpInfo, ClientIP, IpCidr)
| project ClientIP, Info;
NetworkMetrics
| lookup kind=inner sus on ClientIP
| make-series BytesSent=sum(BytesSent) on Timestamp step 1d by Info
| extend (flag, score, baseline) = series_decompose_anomalies(BytesSent)
| extend top_sus = toreal(series_stats_dynamic(score)['max'])
| top 2 by top_sus
```
{{< /note >}}
