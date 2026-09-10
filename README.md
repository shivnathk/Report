# ============================================================
# Azure VM - 90 Day Running Days Report
# Generates CSV report that can be downloaded from Azure Cloud Shell
# ============================================================

# -----------------------------
# CONFIGURATION
# -----------------------------
$SubscriptionId = "<YOUR-SUBSCRIPTION-ID>"
$LookbackDays   = 90

$EndTime   = Get-Date
$StartTime = $EndTime.AddDays(-$LookbackDays)

$OutputFile = "/home/$env:USER/azure-vm-90day-running-report.csv"

# -----------------------------
# LOGIN
# -----------------------------
Connect-AzAccount

# Select subscription
Set-AzContext -SubscriptionId $SubscriptionId

$Subscription = Get-AzSubscription -SubscriptionId $SubscriptionId

Write-Host ""
Write-Host "Subscription : $($Subscription.Name)"
Write-Host "Start Date   : $StartTime"
Write-Host "End Date     : $EndTime"
Write-Host ""

# -----------------------------
# GET ALL VMs
# -----------------------------
$VMs = Get-AzVM -Status

Write-Host "VMs found: $($VMs.Count)"
Write-Host ""

# -----------------------------
# GET ACTIVITY LOG
# -----------------------------
Write-Host "Getting Azure Activity Logs..."

$ActivityLogs = Get-AzActivityLog `
    -StartTime $StartTime `
    -EndTime $EndTime `
    -MaxRecord 100000 `
    -WarningAction SilentlyContinue

Write-Host "Activity log records found: $($ActivityLogs.Count)"
Write-Host ""

# -----------------------------
# PROCESS EACH VM
# -----------------------------
$Results = foreach ($VM in $VMs) {

    Write-Host "Processing VM: $($VM.Name)"

    $VMId = $VM.Id.ToLower()

    # Get VM activity events
    $VMEvents = $ActivityLogs |
        Where-Object {
            $_.ResourceId -and
            $_.ResourceId.ToLower() -eq $VMId
        } |
        Sort-Object EventTimestamp

    # Create event list
    $Events = foreach ($Event in $VMEvents) {

        $Operation = $Event.OperationName.Value

        # VM Start
        if ($Operation -match "start.*virtual machine") {

            [PSCustomObject]@{
                Time  = $Event.EventTimestamp
                State = "Running"
            }
        }

        # VM Stop / Deallocate
        elseif ($Operation -match "deallocate.*virtual machine") {

            [PSCustomObject]@{
                Time  = $Event.EventTimestamp
                State = "Stopped"
            }
        }

        elseif ($Operation -match "poweroff.*virtual machine") {

            [PSCustomObject]@{
                Time  = $Event.EventTimestamp
                State = "Stopped"
            }
        }
    }

    $Events = $Events | Sort-Object Time

    # --------------------------------
    # Calculate running time
    # --------------------------------
    $RunningSeconds = 0
    $RunningStart   = $null

    foreach ($Event in $Events) {

        if ($Event.State -eq "Running") {

            # Start running period
            if ($null -eq $RunningStart) {
                $RunningStart = $Event.Time
            }
        }

        elseif ($Event.State -eq "Stopped") {

            # End running period
            if ($null -ne $RunningStart) {

                $Duration = $Event.Time - $RunningStart

                if ($Duration.TotalSeconds -gt 0) {
                    $RunningSeconds += $Duration.TotalSeconds
                }

                $RunningStart = $null
            }
        }
    }

    # --------------------------------
    # If VM is still running
    # --------------------------------
    $CurrentState = ($VM.Statuses |
        Where-Object {
            $_.Code -like "PowerState/*"
        }).DisplayStatus

    if ($CurrentState -eq "VM running" -and $null -ne $RunningStart) {

        $Duration = $EndTime - $RunningStart

        if ($Duration.TotalSeconds -gt 0) {
            $RunningSeconds += $Duration.TotalSeconds
        }
    }

    # --------------------------------
    # Calculate days/hours
    # --------------------------------
    $RunningDays = $RunningSeconds / 86400
    $RunningHours = $RunningSeconds / 3600

    $StoppedDays = $LookbackDays - $RunningDays

    if ($StoppedDays -lt 0) {
        $StoppedDays = 0
    }

    # --------------------------------
    # Output object
    # --------------------------------
    [PSCustomObject]@{

        SubscriptionName = $Subscription.Name
        SubscriptionId   = $SubscriptionId

        ResourceGroup    = $VM.ResourceGroupName
        VMName           = $VM.Name
        Location         = $VM.Location

        CurrentState     = $CurrentState

        ReportStartDate  = $StartTime
        ReportEndDate    = $EndTime

        RunningDays      = [math]::Round($RunningDays, 2)
        RunningHours     = [math]::Round($RunningHours, 2)

        StoppedDays      = [math]::Round($StoppedDays, 2)

        StartStopEvents  = $Events.Count
    }
}

# -----------------------------
# SORT RESULTS
# -----------------------------
$Results = $Results |
    Sort-Object RunningDays -Descending

# -----------------------------
# EXPORT CSV
# -----------------------------
$Results |
    Export-Csv `
        -Path $OutputFile `
        -NoTypeInformation `
        -Encoding UTF8

# -----------------------------
# DISPLAY REPORT
# -----------------------------
Write-Host ""
Write-Host "==============================================="
Write-Host "REPORT GENERATED SUCCESSFULLY"
Write-Host "==============================================="
Write-Host ""
Write-Host "File:"
Write-Host $OutputFile
Write-Host ""

$Results |
    Format-Table `
        VMName,
        ResourceGroup,
        CurrentState,
        RunningDays,
        StoppedDays,
        StartStopEvents `
        -AutoSize

Write-Host ""
Write-Host "To download:"
Write-Host "Azure Cloud Shell -> Download -> enter:"
Write-Host $OutputFile
Write-Host ""
