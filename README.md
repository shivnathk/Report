VMComputer
| summarize arg_max(TimeGenerated, *) by Computer
| project Computer, BootTime, OperatingSystemFamily, OperatingSystemFullName, TimeGenerated
| order by Computer asc
