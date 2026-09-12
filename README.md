# ============================================================
# Azure VM Charged Usage Report
#
# Gets ALL VMs in a subscription and matches them against
# Azure Cost Management usage data.
#
# Reports:
#   VM Name
#   Resource Group
#   Location
#   VM Size
#   Reporting Days
#   Charged Hours
#   Charged Days
#   Cost
#   Currency
#
# Requirements:
#   Az.Accounts
#   Az.Compute
#
# ============================================================

param (
    [Parameter(Mandatory = $true)]
    [string]$SubscriptionId,

    [Parameter(Mandatory = $false)]
    [int]$Days = 10,

    [Parameter(Mandatory = $false)]
    [string]$OutputCsv = ".\Azure-VM-Charged-Hours.csv"
)

$ErrorActionPreference = "Stop"

# ============================================================
# 1. Connect to Azure
# ============================================================

Write-Host ""
Write-Host "Connecting to Azure..." -ForegroundColor Cyan

Connect-AzAccount -ErrorAction Stop | Out-Null

Set-AzContext `
    -SubscriptionId $SubscriptionId `
    -ErrorAction Stop | Out-Null

Write-Host "Subscription: $SubscriptionId" -ForegroundColor Green

# ============================================================
# 2. Reporting period
# ============================================================

$EndDate = [DateTime]::UtcNow
$StartDate = $EndDate.AddDays(-$Days)

Write-Host ""
Write-Host "Reporting period (UTC)" -ForegroundColor Cyan
Write-Host "Start : $StartDate"
Write-Host "End   : $EndDate"
Write-Host "Days  : $Days"
Write-Host ""

# ============================================================
# 3. Get ALL VMs
# ============================================================

Write-Host "Getting all VMs..." -ForegroundColor Cyan

$vms = Get-AzVM

Write-Host "VMs found: $($vms.Count)" -ForegroundColor Green

if ($vms.Count -eq 0) {
    Write-Host "No VMs found." -ForegroundColor Yellow
    exit
}

# ============================================================
# 4. Build VM lookup
# ============================================================

$vmLookup = @{}

foreach ($vm in $vms) {

    $resourceId = $vm.Id.ToLower()

    $vmLookup[$resourceId] = [PSCustomObject]@{
        VMName          = $vm.Name
        ResourceGroup   = $vm.ResourceGroupName
        Location        = $vm.Location
        VMSize          = $vm.HardwareProfile.VmSize
        ResourceId      = $vm.Id
        ChargedHours    = 0.0
        Cost            = 0.0
        Currency        = ""
    }
}

# ============================================================
# 5. Get Azure access token
# ============================================================

Write-Host ""
Write-Host "Getting Azure access token..." -ForegroundColor Cyan

$token = (Get-AzAccessToken `
    -ResourceUrl "https://management.azure.com/").Token

$headers = @{
    Authorization = "Bearer $token"
    "Content-Type" = "application/json"
}

# ============================================================
# 6. Cost Management API URL
# ============================================================

$scope = "/subscriptions/$SubscriptionId"

$url = "https://management.azure.com$($scope)/providers/Microsoft.CostManagement/query?api-version=2025-03-01"

# ============================================================
# 7. Build Cost Management query
# ============================================================

$body = @{
    type = "Usage"

    timeframe = "Custom"

    timePeriod = @{
        from = $StartDate.ToString("yyyy-MM-ddTHH:mm:ssZ")
        to   = $EndDate.ToString("yyyy-MM-ddTHH:mm:ssZ")
    }

    dataset = @{
        granularity = "Daily"

        aggregation = @{
            totalCost = @{
                name = "PreTaxCost"
                function = "Sum"
            }

            totalQuantity = @{
                name = "UsageQuantity"
                function = "Sum"
            }
        }

        grouping = @(
            @{
                type = "Dimension"
                name = "ResourceId"
            },

            @{
                type = "Dimension"
                name = "Meter"
            },

            @{
                type = "Dimension"
                name = "UnitOfMeasure"
            }
        )
    }
}

$jsonBody = $body | ConvertTo-Json -Depth 10

# ============================================================
# 8. Call Cost Management
# ============================================================

Write-Host ""
Write-Host "Querying Azure Cost Management..." -ForegroundColor Cyan

$response = Invoke-RestMethod `
    -Method Post `
    -Uri $url `
    -Headers $headers `
    -Body $jsonBody

Write-Host "Cost Management query completed." `
    -ForegroundColor Green

# ============================================================
# 9. Process Cost Management results
# ============================================================

$columns = @()

foreach ($column in $response.properties.columns) {
    $columns += $column.name
}

Write-Host ""
Write-Host "Columns returned by Azure:" -ForegroundColor Cyan
Write-Host ($columns -join ", ")
Write-Host ""

# ------------------------------------------------------------
# Find column positions
# ------------------------------------------------------------

$resourceIdIndex = $columns.IndexOf("ResourceId")
$meterIndex      = $columns.IndexOf("Meter")
$unitIndex       = $columns.IndexOf("UnitOfMeasure")
$quantityIndex   = $columns.IndexOf("UsageQuantity")
$costIndex       = $columns.IndexOf("PreTaxCost")

# ============================================================
# 10. Process rows
# ============================================================

if ($response.properties.rows) {

    foreach ($row in $response.properties.rows) {

        if ($resourceIdIndex -lt 0) {
            continue
        }

        $resourceId = [string]$row[$resourceIdIndex]

        if ([string]::IsNullOrWhiteSpace($resourceId)) {
            continue
        }

        $resourceIdLower = $resourceId.ToLower()

        # ----------------------------------------------------
        # Only Azure VM resources
        # ----------------------------------------------------

        if ($resourceIdLower -notmatch `
            "/providers/microsoft.compute/virtualmachines/") {
            continue
        }

        # ----------------------------------------------------
        # Check that this is an hourly meter
        # ----------------------------------------------------

        $meter = ""

        if ($meterIndex -ge 0) {
            $meter = [string]$row[$meterIndex]
        }

        $unit = ""

        if ($unitIndex -ge 0) {
            $unit = [string]$row[$unitIndex]
        }

        # Only process hour-based VM usage
        if ($unit -notmatch "(?i)hour") {
            continue
        }

        # ----------------------------------------------------
        # Find matching VM
        # ----------------------------------------------------

        if (-not $vmLookup.ContainsKey($resourceIdLower)) {
            continue
        }

        $vmRecord = $vmLookup[$resourceIdLower]

        # ----------------------------------------------------
        # Usage quantity
        # ----------------------------------------------------

        if ($quantityIndex -ge 0) {

            if ($null -ne $row[$quantityIndex]) {

                $quantity = [double]$row[$quantityIndex]

                $vmRecord.ChargedHours += $quantity
            }
        }

        # ----------------------------------------------------
        # Cost
        # ----------------------------------------------------

        if ($costIndex -ge 0) {

            if ($null -ne $row[$costIndex]) {

                $cost = [double]$row[$costIndex]

                $vmRecord.Cost += $cost
            }
        }
    }
}

# ============================================================
# 11. Create final report
# ============================================================

$report = foreach ($vmRecord in $vmLookup.Values) {

    $chargedHours = [math]::Round(
        $vmRecord.ChargedHours,
        4
    )

    $chargedDays = [math]::Round(
        ($chargedHours / 24),
        4
    )

    [PSCustomObject]@{

        VMName = $vmRecord.VMName

        ResourceGroup = $vmRecord.ResourceGroup

        Location = $vmRecord.Location

        VMSize = $vmRecord.VMSize

        ReportingDays = $Days

        ChargedHours = $chargedHours

        ChargedDays = $chargedDays

        Cost = [math]::Round(
            $vmRecord.Cost,
            4
        )

        Currency = $vmRecord.Currency

        ResourceId = $vmRecord.ResourceId
    }
}

# ============================================================
# 12. Sort report
# ============================================================

$report = $report | Sort-Object VMName

# ============================================================
# 13. Display
# ============================================================

Write-Host ""
Write-Host "============================================================"
Write-Host "AZURE VM CHARGED HOURS REPORT"
Write-Host "============================================================"
Write-Host ""

$report |
    Format-Table `
        VMName,
        ResourceGroup,
        Location,
        VMSize,
        ReportingDays,
        ChargedHours,
        ChargedDays,
        Cost,
        Currency `
        -AutoSize

# ============================================================
# 14. Export CSV
# ============================================================

$report |
    Export-Csv `
        -Path $OutputCsv `
        -NoTypeInformation `
        -Encoding UTF8

Write-Host ""
Write-Host "============================================================"
Write-Host "REPORT COMPLETED"
Write-Host "============================================================"

Write-Host "Total VMs      : $($report.Count)"
Write-Host "Reporting Days : $Days"
Write-Host "Output CSV     : $OutputCsv" -ForegroundColor Green

Write-Host "============================================================"
