# ============================================================
# Azure VM Charged Hours Report
#
# Purpose:
#   - Get ALL VMs in a subscription
#   - Query Azure Cost Management
#   - Match Cost Management data to VM ResourceId
#   - Identify hourly VM usage
#   - Calculate Charged Hours
#   - Calculate Charged Days
#   - Calculate Cost
#   - Export results to CSV
#
# Requirements:
#   Az.Accounts
#   Az.Compute
#
# Example:
#
# .\Get-AzureVMChargedHours.ps1 `
#     -SubscriptionId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
#     -Days 10
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
# 1. Validate Days
# ============================================================

if ($Days -le 0) {
    throw "Days must be greater than 0."
}

# ============================================================
# 2. Check required modules
# ============================================================

Write-Host ""
Write-Host "Checking PowerShell modules..." -ForegroundColor Cyan

$requiredModules = @(
    "Az.Accounts",
    "Az.Compute"
)

foreach ($module in $requiredModules) {

    if (-not (Get-Module -ListAvailable -Name $module)) {

        Write-Host ""
        Write-Host "Missing module: $module" -ForegroundColor Red

        Write-Host "Install it using:" -ForegroundColor Yellow
        Write-Host "Install-Module $module -Scope CurrentUser"

        throw "Required module $module is not installed."
    }
}

# ============================================================
# 3. Connect to Azure
# ============================================================

Write-Host ""
Write-Host "Connecting to Azure..." -ForegroundColor Cyan

Connect-AzAccount -ErrorAction Stop | Out-Null

# ============================================================
# 4. Set subscription
# ============================================================

Write-Host ""
Write-Host "Setting subscription..." -ForegroundColor Cyan

Set-AzContext `
    -SubscriptionId $SubscriptionId `
    -ErrorAction Stop | Out-Null

$context = Get-AzContext

Write-Host ""
Write-Host "Subscription Name : $($context.Subscription.Name)" `
    -ForegroundColor Green

Write-Host "Subscription ID   : $($context.Subscription.Id)" `
    -ForegroundColor Green

# ============================================================
# 5. Reporting period
# ============================================================

$EndDate = [DateTime]::UtcNow
$StartDate = $EndDate.AddDays(-$Days)

Write-Host ""
Write-Host "============================================================"
Write-Host "REPORTING PERIOD"
Write-Host "============================================================"

Write-Host "Start UTC : $StartDate"
Write-Host "End UTC   : $EndDate"
Write-Host "Days      : $Days"

# ============================================================
# 6. Get ALL VMs
# ============================================================

Write-Host ""
Write-Host "Getting all VMs in subscription..." -ForegroundColor Cyan

$vms = Get-AzVM

Write-Host "Total VMs found: $($vms.Count)" `
    -ForegroundColor Green

if ($vms.Count -eq 0) {

    Write-Host ""
    Write-Host "No VMs found in this subscription." `
        -ForegroundColor Yellow

    exit
}

# ============================================================
# 7. Create VM lookup table
#
# ResourceId is the matching key between:
#
#   Azure Compute
#        and
#   Cost Management
# ============================================================

$vmLookup = @{}

foreach ($vm in $vms) {

    if ([string]::IsNullOrWhiteSpace($vm.Id)) {
        continue
    }

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
# 8. Get Azure Resource Manager token
# ============================================================

Write-Host ""
Write-Host "Getting Azure Resource Manager access token..." `
    -ForegroundColor Cyan

$accessToken = Get-AzAccessToken `
    -ResourceUrl "https://management.azure.com/" `
    -ErrorAction Stop

# ============================================================
# 9. Handle SecureString / normal token
# ============================================================

if ($null -eq $accessToken.Token) {

    throw "Azure access token was not returned."
}

if ($accessToken.Token -is [System.Security.SecureString]) {

    Write-Host "Token returned as SecureString."

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

Write-Host "Azure access token acquired successfully." `
    -ForegroundColor Green

# ============================================================
# 10. Build HTTP headers
# ============================================================

$headers = @{
    Authorization = "Bearer $token"
    "Content-Type" = "application/json"
}

# ============================================================
# 11. Cost Management API URL
# ============================================================

$scope = "/subscriptions/$SubscriptionId"

$url = "https://management.azure.com$($scope)/providers/Microsoft.CostManagement/query?api-version=2025-03-01"

# ============================================================
# 12. Build Cost Management request
# ============================================================

$fromString = $StartDate.ToString(
    "yyyy-MM-ddTHH:mm:ssZ"
)

$toString = $EndDate.ToString(
    "yyyy-MM-ddTHH:mm:ssZ"
)

$body = @{
    type = "Usage"

    timeframe = "Custom"

    timePeriod = @{
        from = $fromString
        to   = $toString
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

            @{
                type = "Dimension"
                name = "Meter"
            }

            @{
                type = "Dimension"
                name = "UnitOfMeasure"
            }
        )
    }
}

$jsonBody = $body | ConvertTo-Json -Depth 20

# ============================================================
# 13. Call Cost Management
# ============================================================

Write-Host ""
Write-Host "Querying Azure Cost Management..." `
    -ForegroundColor Cyan

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
    Write-Host "Cost Management API call failed." `
        -ForegroundColor Red

    Write-Host ""
    Write-Host $_.Exception.Message `
        -ForegroundColor Red

    if ($_.ErrorDetails.Message) {

        Write-Host ""
        Write-Host "Azure response:" `
            -ForegroundColor Yellow

        Write-Host $_.ErrorDetails.Message
    }

    throw
}

Write-Host "Cost Management query completed." `
    -ForegroundColor Green

# ============================================================
# 14. Validate response
# ============================================================

if ($null -eq $response) {

    throw "Cost Management returned an empty response."
}

if ($null -eq $response.properties) {

    throw "Cost Management response does not contain properties."
}

if ($null -eq $response.properties.columns) {

    throw "Cost Management response does not contain columns."
}

# ============================================================
# 15. Read returned columns
# ============================================================

$columns = @()

foreach ($column in $response.properties.columns) {

    $columns += [string]$column.name
}

Write-Host ""
Write-Host "Columns returned by Azure:" `
    -ForegroundColor Cyan

Write-Host ($columns -join ", ")

# ============================================================
# 16. Find column indexes
# ============================================================

$resourceIdIndex = $columns.IndexOf("ResourceId")

$meterIndex = $columns.IndexOf("Meter")

$unitIndex = $columns.IndexOf("UnitOfMeasure")

$quantityIndex = $columns.IndexOf("UsageQuantity")

$costIndex = $columns.IndexOf("PreTaxCost")

# ============================================================
# 17. Validate required columns
# ============================================================

if ($resourceIdIndex -lt 0) {

    throw "ResourceId column was not returned by Cost Management."
}

if ($quantityIndex -lt 0) {

    throw "UsageQuantity column was not returned by Cost Management."
}

if ($costIndex -lt 0) {

    throw "PreTaxCost column was not returned by Cost Management."
}

# ============================================================
# 18. Process Cost Management rows
# ============================================================

Write-Host ""
Write-Host "Matching Cost Management records to VMs..." `
    -ForegroundColor Cyan

$matchedRecords = 0

if ($response.properties.rows) {

    foreach ($row in $response.properties.rows) {

        # ----------------------------------------------------
        # Resource ID
        # ----------------------------------------------------

        $resourceId = [string]$row[$resourceIdIndex]

        if ([string]::IsNullOrWhiteSpace($resourceId)) {
            continue
        }

        $resourceIdLower = $resourceId.ToLowerInvariant()

        # ----------------------------------------------------
        # Only Azure VM resources
        # ----------------------------------------------------

        if ($resourceIdLower -notmatch `
            "/providers/microsoft.compute/virtualmachines/") {

            continue
        }

        # ----------------------------------------------------
        # Meter
        # ----------------------------------------------------

        $meter = ""

        if ($meterIndex -ge 0) {

            $meter = [string]$row[$meterIndex]
        }

        # ----------------------------------------------------
        # Unit
        # ----------------------------------------------------

        $unit = ""

        if ($unitIndex -ge 0) {

            $unit = [string]$row[$unitIndex]
        }

        # ----------------------------------------------------
        # Only hour-based usage
        # ----------------------------------------------------

        if ($unit -notmatch "(?i)hour") {

            continue
        }

        # ----------------------------------------------------
        # Match VM
        # ----------------------------------------------------

        if (-not $vmLookup.ContainsKey($resourceIdLower)) {

            continue
        }

        $vmRecord = $vmLookup[$resourceIdLower]

        # ----------------------------------------------------
        # Quantity
        # ----------------------------------------------------

        if ($null -ne $row[$quantityIndex]) {

            try {

                $quantity = [double]$row[$quantityIndex]

                $vmRecord.ChargedHours += $quantity

            }
            catch {

                Write-Host `
                    "Could not convert quantity for $resourceId" `
                    -ForegroundColor Yellow
            }
        }

        # ----------------------------------------------------
        # Cost
        # ----------------------------------------------------

        if ($null -ne $row[$costIndex]) {

            try {

                $cost = [double]$row[$costIndex]

                $vmRecord.Cost += $cost

            }
            catch {

                Write-Host `
                    "Could not convert cost for $resourceId" `
                    -ForegroundColor Yellow
            }
        }

        $matchedRecords++
    }
}

Write-Host ""
Write-Host "Matched Cost Management records: $matchedRecords" `
    -ForegroundColor Green

# ============================================================
# 19. Build final report
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
# 20. Sort
# ============================================================

$report = $report |
    Sort-Object VMName

# ============================================================
# 21. Display report
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
# 22. Export CSV
# ============================================================

$report |
    Export-Csv `
        -Path $OutputCsv `
        -NoTypeInformation `
        -Encoding UTF8

# ============================================================
# 23. Summary
# ============================================================

$totalChargedHours = (
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

Write-Host "Total VMs       : $($report.Count)"
Write-Host "Reporting Days  : $Days"
Write-Host "Total VM Hours  : $([math]::Round($totalChargedHours, 2))"
Write-Host "Total VM Days   : $([math]::Round(($totalChargedHours / 24), 2))"
Write-Host "Total Cost      : $([math]::Round($totalCost, 2))"
Write-Host ""
Write-Host "CSV              : $OutputCsv" -ForegroundColor Green

Write-Host "============================================================"
