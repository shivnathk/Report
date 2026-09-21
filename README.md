VMComputer
| summarize arg_max(TimeGenerated, *) by Computer
| project
    Computer,
    OperatingSystemFamily,
    OperatingSystemFullName,
    BootTime,
    TimeGenerated
| order by OperatingSystemFamily asc, Computer asc
——————-
Heartbeat
| where TimeGenerated >= ago(90d)
| summarize
    FirstHeartbeat = min(TimeGenerated),
    LastHeartbeat = max(TimeGenerated),
    HeartbeatCount = count(),
    OSType = any(OSType)
    by Computer, VMUUID
| join kind=leftouter (
    VMComputer
    | summarize arg_max(TimeGenerated, *) by VMUUID
    | project
        VMUUID,
        BootTime,
        OperatingSystemFamily,
        OperatingSystemFullName
) on VMUUID
| extend CurrentlyRunning =
    iff(LastHeartbeat >= ago(5m), "Running", "Not Running")
| extend UptimeMinutes =
    iff(
        isnotnull(BootTime) and CurrentlyRunning == "Running",
        datetime_diff("minute", now(), BootTime),
        long(null)
    )
| extend UptimeHours =
    iff(isnotnull(UptimeMinutes),
        round(UptimeMinutes / 60.0, 2),
        real(null))
| extend UptimeDays =
    iff(isnotnull(UptimeMinutes),
        round(UptimeMinutes / 1440.0, 2),
        real(null))
| extend Uptime =
    iff(
        isnotnull(UptimeMinutes),
        strcat(
            toint(UptimeMinutes / 1440), "d ",
            toint((UptimeMinutes % 1440) / 60), "h ",
            UptimeMinutes % 60, "m"
        ),
        "N/A"
    )
| project
    Computer,
    OSType,
    OperatingSystemFullName,
    CurrentlyRunning,
    BootTime,
    Uptime,
    UptimeHours,
    UptimeDays,
    LastHeartbeat,
    FirstHeartbeat,
    HeartbeatCount
| order by Computer asc
————-
Heartbeat
| where TimeGenerated >= ago(1h)
| summarize count() by OSType, Computer
| order by OSType asc, Computer asc
