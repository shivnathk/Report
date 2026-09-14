# ============================================================
# Azure VM Charged Usage Report
#
# Gets:
#   - ALL VMs in subscription
#   - Cost Management usage for last N days
#   - Matches using ResourceId
#   - Cost
#   - Usage quantity
#
# IMPORTANT:
#   Cost Management UsageQuantity is NOT automatically
#   equivalent to VM runtime hours.
#
#   This version keeps the Cost Management grouping valid:
#       ResourceId only
#
#   ChargedHours is capped at the maximum possible hours
#   in the reporting period.
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
    [ValidateRange(1,365)]
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
Write-Host "Subscription    : $($context.Subscription.Name)" `
    -ForegroundColor Green

Write-Host "Subscription ID : $($context.Subscription.Id)" `
    -ForegroundColor Green

# ============================================================
# 3. Date range
# ============================================================

$EndDate = [DateTime]::UtcNow
$StartDate = $EndDate.AddDays(-$Days)

$MaximumHours = $Days * 24

Write-Host ""
Write-Host "Reporting period (UTC)" -ForegroundColor Cyan
Write-Host "Start          : $StartDate"
Write-Host "End            : $EndDate"
Write-Host "Reporting Days : $Days"
Write-Host "Maximum Hours  : $MaximumHours"

# ============================================================
# 4. Get ALL VMs
# ============================================================

Write-Host ""
Write-Host "Getting all VMs..." -ForegroundColor Cyan

$vms = @(Get-AzVM)

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

        UsageQuantity = 0.0

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

$scope = "/subscriptions/$
