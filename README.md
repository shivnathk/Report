Heartbeat
|summarize LastHearbeat=max(TimeGenerated) by Computer
|Join kind=leftouter (
	Event
	|where EventLog == "System"
	|where Source == "Microsoft-Windows-Kernel-General"
	|where EventID == 12
	|summarize LastBoot=max(TimeGenerated) by Computer
) on Computer
|extend UptimeHours = round(datetime_diff('minute',LastHeatbeat,LastBoot) / -60.0,2)
``
---------------------
Heartbeat
| summarize
	FirstHeartbeat=min(TimeGenerated),
	LastHeartbeat=max(TimeGenerated)
	by Computer
| extend UptimeHours = round(datetime_diff('minute',LastHeartbeat, FirstHeartbeat)/60.0,2)
|extend UptimeDays = round(UptimeHours/24.0,2)

----------------------\
|where Namespace == "Computer"
|where Name == "Uptime"
|summarize arg_max(TimeGenerated, Val) by Computer
|project Computer, UptimeHours = round(Val / 3600, 2)
