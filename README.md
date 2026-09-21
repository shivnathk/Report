let HB = 
    Heartbeat
    | where TimeGenerated >= ago(90d)
    | summarize
        FirstHeartbeat = min(TimeGenerated),
        LastHeartbeat = max(TimeGenerated),
        HeartbeatCount = count(),
        OSType = any(OSType)
        by Computer;

let WinBoot =
    Event
    | where TimeGenerated >= ago(90d)
    | where EventID == 6005
    | summarize LastBootTime = max(TimeGenerated) by Computer
    | extend OS = "Windows";

let LinuxBoot =
    Syslog
    | where TimeGenerated >= ago(90d)
    | where SyslogMessage contains "Startup finished"
    | summarize LastBootTime = max(TimeGenerated) by Computer
    | extend OS = "Linux";

let Boots =
    union WinBoot, LinuxBoot
    | summarize LastBootTime = max(LastBootTime) by Computer, OS;

HB
| join kind=leftouter (
    Boots
) on Computer
| extend CurrentlyRunning = 
    iff(LastHeartbeat >= ago(5m), "Running", "Not Running")
| extend UptimeHours =
    iff(
        CurrentlyRunning == "Running" and isnotnull(LastBootTime),
        round(datetime_diff("minute", now(), LastBootTime) / 60.0, 2),
        real(null)
    )
| extend UptimeDays =
    iff(
        CurrentlyRunning == "Running" and isnotnull(LastBootTime),
        round(datetime_diff("minute", now(), LastBootTime) / 1440.0, 2),
        real(null)
    )
| extend Uptime =
    iff(
        CurrentlyRunning == "Running" and isnotnull(LastBootTime),
        strcat(
            toint(datetime_diff("minute", now(), LastBootTime) / 1440),
            "d ",
            toint((datetime_diff("minute", now(), LastBootTime) % 1440) / 60),
            "h ",
            datetime_diff("minute", now(), LastBootTime) % 60,
            "m"
        ),
        "N/A"
    )
| project
    Computer,
    OS,
    CurrentlyRunning,
    LastBootTime,
    Uptime,
    UptimeHours,
    UptimeDays,
    FirstHeartbeat,
    LastHeartbeat,
    HeartbeatCount
| order by UptimeHours asc
