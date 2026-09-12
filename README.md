# ============================================================
# Azure VM Charged Usage Report
#
# Gets:
#   - ALL VMs in subscription
#   - Cost Management usage for last N days
#   - Matches using ResourceId
#   - Charged Hours
#   - Charged Days
#   - Cost
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
# 1. Validate
# ============================================================

if ($Days -le 0) {
    throw "Days must be greater than zero."
}

# ============================================================
# 2. Connect
# ============================================================

Write-Host ""
Write-Host "Connecting to Azure..." -ForegroundColor Cyan

Connect-AzAccount -ErrorAction Stop | Out-Null

Set-AzContext `
    -SubscriptionId $SubscriptionId `
    -ErrorAction Stop | Out-Null

$context = Get-AzContext

Write-Host ""
Write-Host "Subscription : $($context.Subscription.Name)" `
    -ForegroundColor Green

Write-Host "Subscription ID : $($context.Subscription.Id)" `
    -ForegroundColor Green

# ============================================================
# 3. Date range
# ============================================================

$EndDate = [DateTime]::UtcNow
$StartDate = $EndDate.AddDays(-$Days)

Write-Host ""
Write-Host "Reporting period (UTC)" -ForegroundColor Cyan
Write-Host "Start : $StartDate"
Write-Host "End   : $EndDate"
Write-Host "Days  : $Days"

# ============================================================
# 4. Get ALL VMs
# ============================================================

Write-Host ""
Write-Host "Getting all VMs..." -ForegroundColor Cyan

$vms = Get-AzVM

Write-Host "VMs found: $($vms.Count)" -ForegroundColor Green

if ($vms.Count -eq 0) {
    Write-Host "No VMs found." -ForegroundColor Yellow
    exit
}

# ============================================================
# 5. VM lookup
# ============================================================

$vmLookup = @{}

foreach ($vm in $vms) {

    $resourceIdLower = $vm.Id.ToLowerInvariant()

    $vmLookup[$resourceIdLower] = [PSCustomObject]@{

        VMName = $vm.Name

        ResourceGroup = $vm.ResourceGroupName

        Location = $vm.Location

        VMSize = $vm.HardwareProfile.VmSize

        ResourceId = $vm.Id

        ChargedHours = 0.0

        Cost = 0.0

        Currency = ""
    }
}

# ============================================================
# 6. Get ARM token
# ============================================================

Write-Host ""
Write-Host "Getting Azure Resource Manager token..." `
    -ForegroundColor Cyan

$accessToken = Get-AzAccessToken `
    -ResourceUrl "https://management.azure.com/" `
    -ErrorAction Stop

if ($null -eq $accessToken.Token) {
    throw "Could not obtain Azure access token."
}

# Handle SecureString token
if ($accessToken.Token -is [System.Security.SecureString]) {

    $token = [System.Net.NetworkCredential]::new(
        "",
        $accessToken.Token
    ).Password

}
else {

    $token = [string]$accessToken.Token
}

if ([string]::IsNullOrWhiteSpace($token)) {
    throw "Azure access token is empty."
}

$headers = @{
    Authorization = "Bearer $token"
    "Content-Type" = "application/json"
}

Write-Host "Access token acquired." -ForegroundColor Green

# ============================================================
# 7. Cost Management API
# ============================================================

$scope = "/subscriptions/$SubscriptionId"

$url = "https://management.azure.com$($scope)/providers/Microsoft.CostManagement/query?api-version=2025-03-01"

# ============================================================
# 8. IMPORTANT:
#    Group ONLY by ResourceId
#
#    Do NOT group by Meter or UnitOfMeasure.
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
            }
        )
    }
}

$jsonBody = $body | ConvertTo-Json -Depth 20

# ============================================================
# 9. Call Cost Management
# ============================================================

Write-Host ""
Write-Host "Querying Cost Management..." -ForegroundColor Cyan

try {

    $response = Invoke-RestMethod `
        -Method Post `
        -Uri $url `
        -Headers $headers `
        -Body $jsonBody `
        -ErrorAction Stop

}
catch {

    Write-Host ""
    Write-Host "Cost Management API ERROR" -ForegroundColor Red
    Write-Host ""

    Write-Host $_.Exception.Message -ForegroundColor Red

    if ($_.ErrorDetails.Message) {

        Write-Host ""
        Write-Host "Azure response:" -ForegroundColor Yellow
        Write-Host $_.ErrorDetails.Message
    }

    throw
}

Write-Host "Cost Management query successful." `
    -ForegroundColor Green

# ============================================================
# 10. Read columns
# ============================================================

$columns = @()

foreach ($column in $response.properties.columns) {

    $columns += [string]$column.name
}

Write-Host ""
Write-Host "Columns returned:" -ForegroundColor Cyan
Write-Host ($columns -join ", ")

# ============================================================
# 11. Find indexes
# ============================================================

$resourceIdIndex = $columns.IndexOf("ResourceId")

$quantityIndex = $columns.IndexOf("UsageQuantity")

$costIndex = $columns.IndexOf("PreTaxCost")

# ============================================================
# 12. Validate
# ============================================================

if ($resourceIdIndex -lt 0) {
    throw "ResourceId was not returned by Cost Management."
}

if ($quantityIndex -lt 0) {
    throw "UsageQuantity was not returned by Cost Management."
}

if ($costIndex -lt 0) {
    throw "PreTaxCost was not returned by Cost Management."
}

# ============================================================
# 13. Process Cost Management rows
# ============================================================

Write-Host ""
Write-Host "Matching Cost Management data..." `
    -ForegroundColor Cyan

$matchedRecords = 0

if ($response.properties.rows) {

    foreach ($row in $response.properties.rows) {

        $resourceId = [string]$row[$resourceIdIndex]

        if ([string]::IsNullOrWhiteSpace($resourceId)) {
            continue
        }

        $resourceIdLower = $resourceId.ToLowerInvariant()

        # Only VM resources
        if ($resourceIdLower -notmatch `
            "/providers/microsoft.compute/virtualmachines/") {

            continue
        }

        # Match against VM inventory
        if (-not $vmLookup.ContainsKey($resourceIdLower)) {
            continue
        }

        $vmRecord = $vmLookup[$resourceIdLower]

        # ----------------------------------------------------
        # Usage quantity
        # ----------------------------------------------------

        if ($null -ne $row[$quantityIndex]) {

            $quantity = 0.0

            if ([double]::TryParse(
                [string]$row[$quantityIndex],
                [ref]$quantity
            )) {

                $vmRecord.ChargedHours += $quantity
            }
        }

        # ----------------------------------------------------
        # Cost
        # ----------------------------------------------------

        if ($null -ne $row[$costIndex]) {

            $cost = 0.0

            if ([double]::TryParse(
                [string]$row[$costIndex],
                [ref]$cost
            )) {

                $vmRecord.Cost += $cost
            }
        }

        $matchedRecords++
    }
}

Write-Host ""
Write-Host "Matched records: $matchedRecords" `
    -ForegroundColor Green

# ============================================================
# 14. Create report
# ============================================================

$report = foreach ($vmRecord in $vmLookup.Values) {

    $chargedHours = [math]::Round(
        $vmRecord.ChargedHours,
        4
    )

    $chargedDays = [math]::Round(
        $chargedHours / 24,
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
# 15. Sort
# ============================================================

$report = $report |
    Sort-Object VMName

# ============================================================
# 16. Display
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
# 17. Export CSV
# ============================================================

$report |
    Export-Csv `
        -Path $OutputCsv `
        -NoTypeInformation `
        -Encoding UTF8

# ============================================================
# 18. Summary
# ============================================================

$totalHours = (
    $report |
    Measure-Object `
        -Property ChargedHours `
        -Sum
).Sum

$totalCost = (
    $report |
    Measure-Object `
        -Property Cost `
        -Sum
).Sum

Write-Host ""
Write-Host "============================================================"
Write-Host "SUMMARY"
Write-Host "============================================================"

Write-Host "Total VMs      : $($report.Count)"
Write-Host "Reporting Days : $Days"
Write-Host "Total Hours    : $([math]::Round($totalHours, 2))"
Write-Host "Total Days     : $([math]::Round(($totalHours / 24), 2))"
Write-Host "Total Cost     : $([math]::Round($totalCost, 2))"
Write-Host ""
Write-Host "CSV             : $OutputCsv" -ForegroundColor Green

Write-Host "============================================================"
