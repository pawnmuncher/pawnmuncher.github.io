---
title: KQL Detective Season 2
weight: 210
menu:
  notes:
    name: KQL Detective Season 2
    identifier: notes-KQL-Detective-S2
    parent: notes-KQL
    weight: 210
---
{{< note title="Case 0 - Onboarding" >}}

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