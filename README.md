let EndTime = now();
let StartTime = EndTime - 90d;
let BucketSize = 5m;

// Get all VMs seen in the last 90 days
let VMList = materialize(
    Heartbeat
    | where TimeGenerated between (StartTime .. EndTime)
    | extend
        ResourceId = tostring(_ResourceId),
        ResourceGroup = extract(@"/resourceGroups/([^/]+)", 1, tostring(_ResourceId))
    | summarize by Computer, ResourceId, ResourceGroup
);

// Generate 5-minute time buckets for the COMPLETE 90 days
let TimeList = materialize(
    range TimeGenerated from bin(StartTime, BucketSize)
        to bin(EndTime, BucketSize)
        step BucketSize
);

// Get heartbeat presence per VM per 5-minute bucket
let Heartbeat5Min = materialize(
    Heartbeat
    | where TimeGenerated between (StartTime .. EndTime)
    | extend
        ResourceId = tostring(_ResourceId),
        TimeBucket = bin(TimeGenerated, BucketSize)
    | summarize HeartbeatCount = count()
        by Computer, ResourceId, TimeBucket
);

// Create every VM × every 5-minute interval
VMList
| extend JoinKey = 1
| join kind=inner (
    TimeList
    | extend JoinKey = 1
) on JoinKey
| project
    Computer,
    ResourceId,
    ResourceGroup,
    TimeBucket = TimeGenerated
| join kind=leftouter (
    Heartbeat5Min
) on
    Computer,
    ResourceId,
    TimeBucket
| extend HasHeartbeat = iff(isnull(HeartbeatCount), 0, 1)
| summarize
    TotalIntervals = count(),
    UptimeIntervals = countif(HasHeartbeat == 1),
    DowntimeIntervals = countif(HasHeartbeat == 0),
    LastHeartbeat = maxif(TimeBucket, HasHeartbeat == 1)
    by Computer, ResourceGroup, ResourceId
| extend
    ["Total Uptime Hrs"] = round(UptimeIntervals * 5.0 / 60.0, 2),
    ["Total Downtime Hrs"] = round(DowntimeIntervals * 5.0 / 60.0, 2),
    ["5 Min Duration"] = strcat(TotalIntervals * 5, " min"),
    ["Current Running Status"] =
        iff(LastHeartbeat >= EndTime - 10m, "Running", "Not Running")
| project
    ComputerName = Computer,
    ResourceGroup,
    ["Current Running Status"],
    ["Total Uptime Hrs"],
    ["Total Downtime Hrs"],
    ["5 Min Duration"],
    LastHeartbeat
| order by ComputerName asc
