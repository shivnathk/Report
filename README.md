Heartbeat
| where TimeGenerated >= ago(90d)
| summarize
    FirstHeartbeat = min(TimeGenerated),
    LastHeartbeat = max(TimeGenerated),
    HeartbeatCount = count()
    by Computer
| extend HeartbeatHours = round(HeartbeatCount / 60.0, 2)
| extend HeartbeatDays = round(HeartbeatCount / 1440.0, 2)
| extend TotalHoursInPeriod = 90.0 * 24
| extend TotalDaysInPeriod = 90
| project Computer,
          FirstHeartbeat,
          LastHeartbeat,
          HeartbeatHours,
          HeartbeatDays,
          TotalHoursInPeriod,
          TotalDaysInPeriod
| sort by HeartbeatHours asc
