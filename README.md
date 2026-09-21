search *
| where TimeGenerated >= ago(1h)
| summarize Count=count() by $table
| order by Count desc


———-
Heartbeat
| where TimeGenerated >= ago(1h)
| summarize
    LastHeartbeat=max(TimeGenerated),
    Count=count()
    by Computer, OSType
| order by Computer asc

