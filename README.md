let StartTime = ago(90d);
let EndTime = now();
let BucketSize = 5m;

Heartbeat
| where TimeGenerated between (StartTime .. EndTime)
| extend
    ComputerName = Computer,
    ResourceId = tostring(_ResourceId),
    ResourceGroup = extract(
        @"/resourceGroups/([^/]+)",
        1,
        tostring(_ResourceId)
    ),
    TimeBucket = bin(TimeGenerated, BucketSize)
| summarize
    HeartbeatCount = count(),
    LastHeartbeat = max(TimeGenerated)
    by ComputerName, ResourceId, ResourceGroup, TimeBucket
| summarize
    Uptime5Min = countif(HeartbeatCount > 0),
    LastHeartbeat = max(LastHeartbeat)
    by ComputerName, ResourceId, ResourceGroup
| extend
    Total5Min = 25920,
    Downtime5Min = Total5Min - Uptime5Min
| extend
    ["Total Uptime Hrs"] = round(Uptime5Min * 5.0 / 60.0, 2),
    ["Total Downtime Hrs"] = round(Downtime5Min * 5.0 / 60.0, 2),
    ["Current Running Status"] =
        iff(LastHeartbeat >= EndTime - 10m, "Running", "Not Running")
| project
    ComputerName,
    ResourceGroup,
    ["Current Running Status"],
    ["Total Uptime Hrs"],
    ["Total Downtime Hrs"],
    ["5 Min Duration"] = "5 minutes",
    LastHeartbeat
| order by ComputerName asc
