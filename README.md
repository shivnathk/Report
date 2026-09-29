Heartbeat
| extend
    ComputerName = Computer,
    ResourceId = tostring(_ResourceId),
    ResourceGroup = extract(
        @"/resourceGroups/([^/]+)",
        1,
        tostring(_ResourceId)
    )
| summarize
    FirstEverHeartbeat = min(TimeGenerated)
    by ComputerName, ResourceId, ResourceGroup
| join kind=inner (
    Heartbeat
    | where TimeGenerated >= ago(90d)
    | extend
        ComputerName = Computer,
        ResourceId = tostring(_ResourceId),
        ResourceGroup = extract(
            @"/resourceGroups/([^/]+)",
            1,
            tostring(_ResourceId)
        ),
        TimeBucket = bin(TimeGenerated, 5m)
    | summarize
        HeartbeatCount = count(),
        LastHeartbeat = max(TimeGenerated)
        by ComputerName, ResourceId, ResourceGroup, TimeBucket
) on ComputerName, ResourceId, ResourceGroup
| summarize
    FirstEverHeartbeat = min(FirstEverHeartbeat),
    LastHeartbeat = max(LastHeartbeat),
    Uptime5MinIntervals = count()
    by ComputerName, ResourceId, ResourceGroup
| extend
    MeasurementStart =
        iff(
            FirstEverHeartbeat > ago(90d),
            FirstEverHeartbeat,
            ago(90d)
        )
| extend
    MeasurementMinutes =
        datetime_diff("minute", now(), MeasurementStart)
| extend
    Total5MinIntervals =
        toint(ceiling(MeasurementMinutes / 5.0))
| extend
    Downtime5MinIntervals =
        Total5MinIntervals - Uptime5MinIntervals
| extend
    ["Total Uptime Hrs"] =
        round(Uptime5MinIntervals * 5.0 / 60.0, 2),
    ["Total Downtime Hrs"] =
        round(Downtime5MinIntervals * 5.0 / 60.0, 2),
    ["Current Running Status"] =
        iff(LastHeartbeat >= ago(10m), "Running", "Not Running")
| project
    ComputerName,
    ResourceGroup,
    ["Current Running Status"],
    ["Total Uptime Hrs"],
    ["Total Downtime Hrs"],
    ["5 Min Duration"] = "5 minutes",
    ["First Heartbeat"] = FirstEverHeartbeat,
    ["Last Heartbeat"] = LastHeartbeat
| order by ComputerName asc



——————————————-
