# ============================================================
# Azure VM Charged Hours Report
#
# Gets ALL VMs in a subscription and matches them against
# Azure Cost Management usage data.
#
# Report columns:
#   VMName
#   ResourceGroup
#   Location
#   VMSize
#   ReportingDays
#   ChargedHours
#   ChargedDays
#   Cost
#   Currency
#
# Requirements:
#   Az.Accounts
#   Az.Compute
#   Az.CostManagement
#
# Example:
#
# .\Get-AzureVMChargedHours.ps1 `
#     -SubscriptionId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
#     -Days 10
# ============================================================

param(
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
Write-Host "Reporting Period (UTC)" -ForegroundColor Cyan
Write-Host "Start : $StartDate"
Write-Host "End   : $EndDate"
Write-Host "Days  : $Days"
Write-Host ""

# ============================================================
# 3. Get ALL VMs
# ============================================================

Write-Host "Getting all VMs from Azure Compute..." -ForegroundColor Cyan

$vms = Get-AzVM

Write-Host "VMs found: $($vms.Count)" -ForegroundColor Green

if ($vms.Count -eq 0) {

    Write-Host ""
    Write-Host "No VMs found in this subscription." `
        -ForegroundColor Yellow

    exit
}

# ============================================================
# 4. Create VM lookup table
# ============================================================

$vmLookup = @{}

foreach ($vm in $vms) {

    $resourceId = $vm.Id.ToLower()

    $vmLookup[$resourceId] = [PSCustomObject]@{

        VMName        = $vm.Name
        ResourceGroup = $vm.ResourceGroupName
        Location      = $vm.Location
        VMSize        = $vm.HardwareProfile.VmSize
        ResourceId    = $vm.Id

        ChargedQuantity = 0
        Cost            = 0
        Currency        = ""
    }
}

# ============================================================
# 5. Query Azure Cost Management
# ============================================================

Write-Host ""
Write-Host "Querying Azure Cost Management..." -ForegroundColor Cyan

$scope = "/subscriptions/$SubscriptionId"

$grouping = @(
    New-AzCostManagementQueryGroupingObject `
        -Type Dimension `
        -Name "ResourceId"
)

$aggregation = @{

    totalQuantity = @{
        Name     = "UsageQuantity"
        Function = "Sum"
    }

    totalCost = @{
        Name     = "PreTaxCost"
        Function = "Sum"
    }
}

$result = Invoke-AzCostManagementQuery `
    -Scope $scope `
    -Type "Usage" `
    -Timeframe "Custom" `
    -TimePeriodFrom $StartDate `
    -TimePeriodTo $EndDate `
    -DatasetGranularity "Daily" `
    -DatasetGrouping $grouping `
    -DatasetAggregation $aggregation

Write-Host "Cost Management query completed." `
    -ForegroundColor Green

# ============================================================
# 6. Identify Cost Management columns
# ============================================================

$resourceIdIndex = -1
$quantityIndex   = -1
$costIndex       = -1
$currencyIndex   = -1

for ($i = 0; $i -lt $result.Column.Count; $i++) {

    switch ($result.Column[$i].Name) {

        "ResourceId" {
            $resourceIdIndex = $i
        }

        "UsageQuantity" {
            $quantityIndex = $i
        }

        "PreTaxCost" {
            $costIndex = $i
        }

        "Currency" {
            $currencyIndex = $i
        }
    }
}

# ============================================================
# 7. Match Cost Management data to VMs
# ============================================================

Write-Host ""
Write-Host "Matching Cost Management records to VMs..." `
    -ForegroundColor Cyan

if ($result.Row) {

    foreach ($row in $result.Row) {

        if ($resourceIdIndex -lt 0) {
            continue
        }

        $resourceId = [string]$row[$resourceIdIndex]

        if ([string]::IsNullOrWhiteSpace($resourceId)) {
            continue
        }

        $resourceIdLower = $resourceId.ToLower()

        # Only Azure VM resources
        if ($resourceIdLower -notmatch `
            "/providers/microsoft\.compute/virtualmachines/") {

            continue
        }

        # Match with VM inventory
        if ($vmLookup.ContainsKey($resourceIdLower)) {

            $vmRecord = $vmLookup[$resourceIdLower]

            # Usage quantity
            if ($quantityIndex -ge 0 -and
                $null -ne $row[$quantityIndex]) {

                $vmRecord.ChargedQuantity += `
                    [double]$row[$quantityIndex]
            }

            # Cost
            if ($costIndex -ge 0 -and
                $null -ne $row[$costIndex]) {

                $vmRecord.Cost += `
                    [double]$row[$costIndex]
            }

            # Currency
            if ($currencyIndex -ge 0 -and
                $null -ne $row[$currencyIndex]) {

                $vmRecord.Currency = `
                    [string]$row[$currencyIndex]
            }
        }
    }
}

# ============================================================
# 8. Create final report
# ============================================================

$report = foreach ($vmRecord in $vmLookup.Values) {

    # Quantity returned by Cost Management
    $chargedHours = [math]::Round(
        $vmRecord.ChargedQuantity,
        4
    )

    # Convert hours to days
    $chargedDays = [math]::Round(
        ($chargedHours / 24),
        4
    )

    [PSCustomObject]@{

        VMName        = $vmRecord.VMName

        ResourceGroup = $vmRecord.ResourceGroup

        Location      = $vmRecord.Location

        VMSize        = $vmRecord.VMSize

        # Requested reporting window
        ReportingDays = $Days

        # Azure usage quantity
        ChargedHours  = $chargedHours

        # ChargedHours / 24
        ChargedDays   = $chargedDays

        Cost = [math]::Round(
            $vmRecord.Cost,
            4
        )

        Currency = $vmRecord.Currency

        ResourceId = $vmRecord.ResourceId
    }
}

$report = $report |
    Sort-Object VMName

# ============================================================
# 9. Display report
# ============================================================

Write-Host ""
Write-Host "============================================================"
Write-Host "AZURE VM CHARGED USAGE REPORT"
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
# 10. Export CSV
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
Write-Host "CSV File       : $OutputCsv" -ForegroundColor Green

Write-Host "============================================================"
