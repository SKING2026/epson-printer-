# ===== Epson Printer Share Fix =====
# Change this to your Epson printer name exactly as shown in Devices and Printers
$printerName = "EPSON XP-7100 Series"

# 1) Make sure required services are running
$services = "Spooler","LanmanServer","LanmanWorkstation"
foreach ($svc in $services) {
    try {
        $s = Get-Service -Name $svc -ErrorAction Stop
        if ($s.Status -ne "Running") {
            Start-Service -Name $svc
            Set-Service -Name $svc -StartupType Automatic
        }
        Write-Host "$svc is running." -ForegroundColor Green
    }
    catch {
        Write-Host "Service not found or failed: $svc" -ForegroundColor Red
    }
}

# 2) Enable Windows printer sharing firewall rules
try {
    Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing" -ErrorAction SilentlyContinue
    Write-Host "File and Printer Sharing firewall rules enabled." -ForegroundColor Green
}
catch {
    Write-Host "Could not enable File and Printer Sharing rule." -ForegroundColor Yellow
}

# 3) Check if the printer exists
$printer = Get-Printer -Name $printerName -ErrorAction SilentlyContinue
if (-not $printer) {
    Write-Host "Printer '$printerName' not found. Check the exact printer name." -ForegroundColor Red
    exit
}

# 4) Share the printer
$shareName = "EpsonPrinter"

if (-not $printer.Shared) {
    Set-Printer -Name $printerName -Shared $true -ShareName $shareName
    Write-Host "Printer shared successfully as $shareName" -ForegroundColor Green
}
else {
    Set-Printer -Name $printerName -ShareName $shareName
    Write-Host "Printer is already shared. Share name set to $shareName" -ForegroundColor Yellow
}

# 5) Clear stuck print queue if needed
try {
    Clear-PrinterQueue -Name $printerName -ErrorAction SilentlyContinue
    Write-Host "Print queue cleared." -ForegroundColor Green
}
catch {
    Write-Host "No queue to clear or queue clear failed." -ForegroundColor Yellow
}

# 6) Verify final result
Get-Printer -Name $printerName | Format-List Name,Shared,ShareName,DriverName,PortName
Get-SmbShare | Where-Object { $_.Name -match "Epson" } | Format-Table Name,Path,Description

Write-Host ""
Write-Host "Done. Try accessing this printer from another PC on the network." -ForegroundColor Cyan
