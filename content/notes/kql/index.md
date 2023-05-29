---
title: KQL Notes
menu:
  notes:
    name: KQL
    identifier: notes-KQL
    weight: 0
---

# KQL Notes
https://detective.kusto.io/inbox

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

```

{{< /note >}}

{{< note title="Kusto Detective KQL Challenge Information" >}}

```SQL
Welcome to the Kusto Detective Agency - Season 2!
If you've been here before, congrats on leveling up! Click that log-in button to access your account.

But if you're new and don't have a free Kusto cluster or KQL database, fear not! Follow the instructions in the FAQ section to create one and join the detective party. Your shiny new cluster will be your go-to investigative tool, and its URL will be your secret agent identity.

If you have been here for Season 1, you may be surprised to find yourself as a Rookie again. You see, it's all about innovation and hitting refresh. So, it's a fresh start for everyone. Yet we believe in excellence and that's why we need your detective skills to unveil the crème de la crème of detectives from the past year, 2022. This is like the ultimate leaderboard challenge where we crown the "Most Epic Detective of the Year." Exciting, right?

Imagine our agency as a buzzing beehive, like StackOverflow on steroids. We have a crazy number of cases popping up every day, each with a juicy bounty attached (yes, cold, hard cash!). And guess what? We've got thousands of Kusto Detectives scattered across the globe, all itching to pick a case and earn their detective stripes. But here's the catch: only the first detective to crack the case gets the bounty and major street cred!

So, your mission, should you choose to accept it, is to dig into the vast archives of our system operation logs from the legendary year 2022. You're on a quest to unearth the absolute legend, the detective with the biggest impact on our business—the one who raked in the most moolah by claiming bounties like a boss!

Feeling a bit rusty or want to level up your Kusto skills? No worries, my friend. We've got your back with the "Train Me" section. It's like a power-up that'll help you sharpen your Kusto-fu to tackle each case head-on. Oh, and if you stumble upon a mind-boggling case and need a little nudge, the "Hints" are there to save the day!

Now, strap on your detective hat, embrace the thrill, and get ready to rock this investigation. The fate of the "Most Epic Detective of the Year" rests in your hands!

Good luck, rookie, and remember to bring your sense of humor along for this wild ride!

Lieutenant Laughter
```

{{< /note >}}