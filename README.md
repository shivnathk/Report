let Lookback = 90d;
let HeartbeatInterval = 1m;
let StopThreshold = 5m;

// Get all heartbeats in the last 90 days
Heartbeat
| where TimeGenerated >= ago(Lookback)
| project Computer, TimeGenerated
| sort by Computer asc, TimeGenerated asc

// Find the previous heartbeat for each VM
| serialize
| extend PreviousHeartbeat = prev(TimeGenerated)
| extend PreviousComputer = prev(Computer)

// Identify the start of a new running period
| extend NewRun =
    iff(
        Computer != PreviousComputer
        or isempty(PreviousHeartbeat)
        or TimeGenerated - PreviousHeartbeat > StopThreshold,
        1,
        0
    )

// Create a running-period ID
| extend RunId = row_cumsum(NewRun)

// Calculate each running period
| summarize
    RunStart = min(TimeGenerated),
    RunEnd = max(TimeGenerated),
    Heartbeats = count()
    by Computer, RunId

// A heartbeat period represents actual observed running time.
// Add the heartbeat interval to the last heartbeat so the final
// heartbeat contributes to the running interval.
| extend RunEndWithInterval = RunEnd + HeartbeatInterval

// Do not allow the calculated interval to extend beyond "now"
| extend RunEndWithInterval =
    iff(RunEndWithInterval > now(), now(), RunEndWithInterval)

// Calculate uptime for each individual running period
| extend RunningHours =
    datetime_diff("second", RunEndWithInterval, RunStart) / 3600.0

// Summarize all running periods for each VM
| summarize
    FirstHeartbeat = min(RunStart),
    LastHeartbeat = max(RunEnd),
    RunningHours = round(sum(RunningHours), 2),
    RunningDays = round(sum(RunningHours) / 24.0, 2),
    RunningPeriods = count()
    by Computer

// Calculate total observation period
| extend TotalHours = round(datetime_diff("hour", now(), ago(Lookback)) * -1.0, 2)
| extend TotalDays = round(TotalHours / 24.0, 2)

// Current state
| extend CurrentRunning =
    iff(LastHeartbeat >= ago(StopThreshold), "Running", "Not Running")

| project
    Computer,
    FirstHeartbeat,
    LastHeartbeat,
    CurrentRunning,
    RunningPeriods,
    RunningHours,
    RunningDays,
    TotalHours,
    TotalDays

| sort by RunningHours asc
