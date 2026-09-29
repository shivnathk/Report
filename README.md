let EndTime = now();
let StartTime = EndTime - 90d;
let Bucket = 5m;

// All VMs seen during the 90-day period
let VMs =
    Heartbeat
    | where TimeGenerated between (StartTime .. EndTime)
    | extend ResourceId = tostring(_ResourceId)
    | extend ResourceGroup = extract(
        @"/resourceGroups/([^/]+)",
        1,
        ResourceId
    )
    | summarize by Computer, ResourceId, ResourceGroup;

// Generate exactly 5-minute intervals for the complete 90 days
let TimeBuckets =
    range TimeGenerated from bin(StartTime, Bucket)
        to bin(EndTime, Bucket)
        step Bucket;

// Heartbeat presence in each 5-minute interval
let HB =
    Heartbeat
    | where TimeGenerated between (StartTime .. EndTime)
    | extend
        ResourceId = tostring(_ResourceId),
        TimeBucket = bin(TimeGenerated, Bucket)
    | summarize HeartbeatCount = count()
        by Computer, ResourceId, TimeBucket;

// Create complete VM × 5-minute interval matrix
VMs
| extend JoinKey = 1
| join kind=inner (
    TimeBuckets
    | extend JoinKey = 1
) on JoinKey
| project
    Computer,
    ResourceId,
    ResourceGroup,
    TimeGenerated
| join kind=leftouter (
    HB
) on
    Computer,
    ResourceId,
    $left.TimeGenerated == $right.TimeBucket
| extend HasHeartbeat = iff(isnotnull(HeartbeatCount), 1, 0)
| summarize
    Total5MinIntervals = count(),
    Uptime5MinIntervals = countif(HasHeartbeat == 1),
    Downtime5MinIntervals = countif(HasHeartbeat == 0),
    LastHeartbeat = maxif(TimeGenerated, HasHeartbeat == 1)
    by Computer, ResourceGroup, ResourceId
| extend
    ["Total Uptime Hrs"] =
        round(Uptime5MinIntervals * 5.0 / 60.0, 2),
    ["Total Downtime Hrs"] =
        round(Downtime5MinIntervals * 5.0 / 60.0, 2),
    ["5 Min Duration"] =
        strcat(Total5MinIntervals * 5, " min"),
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
