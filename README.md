Event
| where TimeGenerated >= ago(90d)
| where EventID == 6005
| summarize LastBootTime = max(TimeGenerated) by Computer
| extend UptimeHours = round(datetime_diff('minute', now(), LastBootTime) / 60.0, 2)
| extend UptimeDays = round(datetime_diff('minute', now(), LastBootTime) / 1440.0, 2)
| extend Uptime = strcat(
    toint(datetime_diff('minute', now(), LastBootTime) / 1440), "d ",
    toint((datetime_diff('minute', now(), LastBootTime) % 1440) / 60), "h ",
    datetime_diff('minute', now(), LastBootTime) % 60, "m"
)
| project Computer, LastBootTime, Uptime, UptimeHours, UptimeDays
| order by UptimeHours asc


———————————
Syslog
| where TimeGenerated >= ago(90d)
| where SyslogMessage contains "Startup finished"
| summarize LastBootTime = max(TimeGenerated) by Computer
| extend UptimeHours = round(datetime_diff('minute', now(), LastBootTime) / 60.0, 2)
| extend UptimeDays = round(datetime_diff('minute', now(), LastBootTime) / 1440.0, 2)
| extend Uptime = strcat(
    toint(datetime_diff('minute', now(), LastBootTime) / 1440), "d ",
    toint((datetime_diff('minute', now(), LastBootTime) % 1440) / 60), "h ",
    datetime_diff('minute', now(), LastBootTime) % 60, "m"
)
| project Computer, LastBootTime, Uptime, UptimeHours, UptimeDays
| order by UptimeHours asc
