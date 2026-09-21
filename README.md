let HeartbeatData =
    Heartbeat
    | where TimeGenerated >= ago(90d)
    | summarize
        FirstHeartbeat = min(TimeGenerated),
        LastHeartbeat = max(TimeGenerated),
        HeartbeatCount = count(),
        OSType = any(OSType)
        by Computer;

//
// Windows boot time
// Event ID 6005 = Event Log service started, normally occurring during boot
//
let WindowsBoot =
    Event
    | where TimeGenerated >= ago(90d)
    | where EventID == 6005
    | summarize LastBootTime = max(TimeGenerated) by Computer;

//
// Linux boot time
// systemd writes "Startup finished" during system startup
//
let LinuxBoot =
    Syslog
    | where TimeGenerated >= ago(90d)
    | where SyslogMessage has "Startup finished"
    | summarize LastBootTime = max(TimeGenerated) by Computer;

//
// Combine Windows and Linux boot information
//
let BootData =
    union
        (WindowsBoot | extend OS = "Windows"),
        (LinuxBoot | extend OS = "Linux")
    | summarize LastBootTime = max(LastBootTime) by Computer, OS;

HeartbeatData
| join kind=leftouter BootData on Computer
| extend CurrentlyRunning =
    iff(LastHeartbeat >= ago(5m), "Running", "Not Running")
| extend
    UptimeHours =
        iff(
            CurrentlyRunning == "Running" and isnotnull(LastBootTime),
            round((now() - LastBootTime) / 1h, 2),
            real(null)
        )
| extend
    UptimeDays =
        iff(
            CurrentlyRunning == "Running" and isnotnull(LastBootTime),
            round((now() - LastBootTime) / 1d, 2),
            real(null)
        )
| extend
    Uptime =
        iff(
            CurrentlyRunning == "Running" and isnotnull(LastBootTime),
            strcat(
                toint((now() - LastBootTime) / 1d),
                "d ",
                toint(((now() - LastBootTime) % 1d) / 1h),
                "h ",
                toint(((now() - LastBootTime) % 1h) / 1m),
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
| sort by UptimeHours asc
