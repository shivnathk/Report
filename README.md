Heartbeat
| where TimeGenerated >= ago(90d)
| summarize
    FirstHeartbeat = min(TimeGenerated),
    LastHeartbeat = max(TimeGenerated),
    HeartbeatCount = count()
    by Computer
| extend RunningHours = round(HeartbeatCount / 60.0, 2)
| extend RunningDays = round(HeartbeatCount / 1440.0, 2)
| extend TotalHours = 90.0 * 24
| extend TotalDays = 90
| extend CurrentRunning = iff(LastHeartbeat >= ago(5m), "Running" , "Not Running")
| project Computer,
          FirstHeartbeat,
          LastHeartbeat,
		  CurrentRunning,
          RunningHours,
          RunningDays,
          TotalHours,
          TotalDays
| sort by RunningHours asc
