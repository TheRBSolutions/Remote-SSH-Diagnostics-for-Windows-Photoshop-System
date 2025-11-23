# Remote-SSH-Diagnostics-for-Windows-Photoshop-System

# YES! Complete SSH Remote Diagnostics Guide

**All diagnostics CAN be done via SSH while designer works on Photoshop**

**Project Complexity: High**
**Estimated Days: 5-6 days** (3-4 hours/day)
**Your Rate: $3/hour**

## Budget Breakdown:

**Total Hours: 15-24 hours**
- **Minimum**: 15 hours × $3 = **$45 USD / ₹3,780 INR**
- **Maximum**: 24 hours × $3 = **$72 USD / ₹6,048 INR**

---

## Complete SSH Remote Workflow

### Prerequisites Setup (One-time, 1-2 hours)

#### Step 1: Designer Enables SSH on Windows

**Designer runs once (as Administrator):**

```powershell
# Install OpenSSH Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Start and enable
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'

# Allow through firewall
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22

# Get IP address to share with you
ipconfig | findstr IPv4
```

#### Step 2: You Connect from Your System

```bash
# From your Linux/Mac/Windows terminal
ssh designer_username@192.168.1.XXX

# Enter password when prompted
# You're now on designer's Windows machine!
```

---

## Day 1: SSH-Based System Diagnostics (3-4 hours)

### 1.1 Initial Connection & Setup (30 min)

```bash
# Connect
ssh designer@192.168.1.XXX

# Switch to PowerShell (better for Windows diagnostics)
powershell.exe

# Set execution policy for current session
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

# Create working directories
New-Item -ItemType Directory -Path "C:\RemoteDiag" -Force
New-Item -ItemType Directory -Path "C:\RemoteDiag\Scripts" -Force
New-Item -ItemType Directory -Path "C:\RemoteDiag\Output" -Force
cd C:\RemoteDiag
```

### 1.2 Check Current Photoshop Status (10 min)

```powershell
# Is Photoshop running?
$ps = Get-Process Photoshop -ErrorAction SilentlyContinue
if ($ps) {
    Write-Host "✓ Photoshop is running (PID: $($ps.Id))" -ForegroundColor Green
    Write-Host "  RAM: $([math]::Round($ps.WorkingSet64/1GB, 2)) GB"
    Write-Host "  CPU Time: $([math]::Round($ps.CPU, 2))s"
    Write-Host "  Threads: $($ps.Threads.Count)"
} else {
    Write-Host "❌ Photoshop not running. Ask designer to launch it." -ForegroundColor Red
}

# System overview
$os = Get-CimInstance Win32_OperatingSystem
$cpu = Get-CimInstance Win32_Processor | Select-Object -First 1
$gpu = Get-CimInstance Win32_VideoController | Where-Object { $_.Name -notlike "*Microsoft*" }

Write-Host "`nSystem Info:"
Write-Host "CPU: $($cpu.Name)"
Write-Host "RAM: $([math]::Round($os.TotalVisibleMemorySize/1MB, 2)) GB"
Write-Host "Free RAM: $([math]::Round($os.FreePhysicalMemory/1MB, 2)) GB"
Write-Host "GPU: $($gpu.Name)"
```

### 1.3 Upload Diagnostic Scripts via SSH (15 min)

**From your local machine (separate terminal):**

```bash
# Upload the master diagnostic script
scp PS_MasterDiagnostics.ps1 designer@192.168.1.XXX:C:/RemoteDiag/Scripts/

# Upload GPU diagnostics
scp GPU_DeepDiag.ps1 designer@192.168.1.XXX:C:/RemoteDiag/Scripts/

# Upload monitoring script
scp PS_Monitor.ps1 designer@192.168.1.XXX:C:/RemoteDiag/Scripts/

# Or upload all at once
scp *.ps1 designer@192.168.1.XXX:C:/RemoteDiag/Scripts/
```

**Alternative: Create scripts directly via SSH:**

```powershell
# In SSH session, create script inline
@'
# Your script content here
Get-Process Photoshop | Select-Object CPU, WorkingSet64
'@ | Out-File -FilePath "C:\RemoteDiag\Scripts\test.ps1"

# Run it
powershell.exe -ExecutionPolicy Bypass -File "C:\RemoteDiag\Scripts\test.ps1"
```

### 1.4 Run Performance Monitor Collection (1 hour)

```powershell
# Start Performance Monitor data collection
$collectorName = "PhotoshopRemote"

# Create data collector
logman create counter $collectorName -f bincirc -v mmddhhmm -max 500 -c `
"\Processor(*)\% Processor Time" `
"\Process(Photoshop)\*" `
"\GPU Engine(*)\*" `
"\PhysicalDisk(*)\*" `
"\Memory\*" `
-si 00:00:01 -o "C:\RemoteDiag\Output\perfmon.blg"

# Start collection
logman start $collectorName

Write-Host "✓ Performance monitoring started" -ForegroundColor Green
Write-Host "Designer can work normally now. Let this run for 10-15 minutes."
Write-Host "Type 'logman stop $collectorName' when ready to analyze."
```

**Designer works in Photoshop for 10-15 minutes while you monitor remotely...**

### 1.5 Real-Time Monitoring (While Designer Works) (1 hour)

```powershell
# Live monitoring loop (run in SSH session)
Write-Host "Real-time Photoshop monitoring (Press Ctrl+C to stop)" -ForegroundColor Cyan

while ($true) {
    $ps = Get-Process Photoshop -ErrorAction SilentlyContinue
    
    if ($ps) {
        # Get metrics
        $cpu = (Get-Counter "\Process(Photoshop)\% Processor Time" -ErrorAction SilentlyContinue).CounterSamples.CookedValue
        $ramGB = [math]::Round($ps.WorkingSet64 / 1GB, 2)
        $pageFaults = (Get-Counter "\Process(Photoshop)\Page Faults/sec" -ErrorAction SilentlyContinue).CounterSamples.CookedValue
        $diskQueue = (Get-Counter "\PhysicalDisk(_Total)\Current Disk Queue Length" -ErrorAction SilentlyContinue).CounterSamples.CookedValue
        
        # GPU (if available)
        $gpuPct = 0
        try {
            $gpuCounter = Get-Counter "\GPU Engine(*engtype_3D)\Utilization Percentage" -ErrorAction Stop
            $gpuPct = [math]::Round(($gpuCounter.CounterSamples | Measure-Object CookedValue -Average).Average, 2)
        } catch {}
        
        $timestamp = Get-Date -Format "HH:mm:ss"
        
        # Color coding
        $cpuColor = if ($cpu -gt 90) { "Red" } elseif ($cpu -gt 70) { "Yellow" } else { "Green" }
        $pfColor = if ($pageFaults -gt 100) { "Red" } elseif ($pageFaults -gt 50) { "Yellow" } else { "Green" }
        
        # Display
        Clear-Host
        Write-Host "=== Photoshop Live Monitor ===" -ForegroundColor Cyan
        Write-Host "Time: $timestamp" -ForegroundColor Gray
        Write-Host ""
        Write-Host "CPU:        " -NoNewline
        Write-Host "$([math]::Round($cpu,1))%" -ForegroundColor $cpuColor
        Write-Host "RAM:        $ramGB GB"
        Write-Host "Page Faults: " -NoNewline
        Write-Host "$([math]::Round($pageFaults,1))/sec" -ForegroundColor $pfColor
        Write-Host "Disk Queue: $([math]::Round($diskQueue,1))"
        Write-Host "GPU:        $gpuPct%"
        Write-Host ""
        Write-Host "Press Ctrl+C to stop monitoring" -ForegroundColor Gray
    } else {
        Write-Host "Photoshop not running" -ForegroundColor Red
    }
    
    Start-Sleep -Seconds 2
}
```

### 1.6 Stop Collection & Export Data (30 min)

```powershell
# Stop performance monitor
logman stop PhotoshopRemote
logman delete PhotoshopRemote

# Export to CSV for analysis
# You can do this remotely OR download the .blg file
```

---

## Day 2: Advanced Process Analysis via SSH (4-5 hours)

### 2.1 Download & Install Tools Remotely (1 hour)

```powershell
# Download Sysinternals tools directly from Microsoft
$toolsPath = "C:\RemoteDiag\Tools"
New-Item -ItemType Directory -Path $toolsPath -Force

# Download Process Explorer
Invoke-WebRequest -Uri "https://live.sysinternals.com/procexp64.exe" -OutFile "$toolsPath\procexp64.exe"

# Download Process Monitor
Invoke-WebRequest -Uri "https://live.sysinternals.com/Procmon64.exe" -OutFile "$toolsPath\Procmon64.exe"

# Download RAMMap
Invoke-WebRequest -Uri "https://live.sysinternals.com/RAMMap.exe" -OutFile "$toolsPath\RAMMap.exe"

# Download Handle
Invoke-WebRequest -Uri "https://live.sysinternals.com/handle64.exe" -OutFile "$toolsPath\handle64.exe"

# Download GPU-Z (portable)
Invoke-WebRequest -Uri "https://ftp.nluug.nl/pub/games/PC/guru3d/gpu-z/GPU-Z.2.55.0.exe" -OutFile "$toolsPath\GPU-Z.exe"

# Accept EULA for all Sysinternals tools
Get-ChildItem "$toolsPath\*.exe" | ForEach-Object { 
    Start-Process $_.FullName -ArgumentList "/accepteula" -Wait -WindowStyle Hidden
}

Write-Host "✓ Tools downloaded and configured" -ForegroundColor Green
```

### 2.2 Run Process Monitor Capture (1 hour)

**Problem:** Process Monitor is GUI-based, but we can use it in CLI mode!

```powershell
# Configure Process Monitor for command-line capture
$procmonPath = "C:\RemoteDiag\Tools\Procmon64.exe"
$outputPML = "C:\RemoteDiag\Output\photoshop_capture.pml"
$outputCSV = "C:\RemoteDiag\Output\photoshop_capture.csv"

# Create filter file
$filterXML = @"
<?xml version="1.0" encoding="UTF-8"?>
<ProcessMonitorFilter>
    <Rule>
        <ProcessName>Photoshop.exe</ProcessName>
        <Relation>is</Relation>
        <Action>Include</Action>
    </Rule>
    <Rule>
        <Duration>0.050</Duration>
        <Relation>is greater than</Relation>
        <Action>Include</Action>
    </Rule>
</ProcessMonitorFilter>
"@

$filterXML | Out-File -FilePath "C:\RemoteDiag\ps_filter.pmf" -Encoding UTF8

# Start Process Monitor in background (minimized, no GUI interaction needed)
Write-Host "Starting Process Monitor capture..." -ForegroundColor Yellow
Write-Host "Tell designer to work normally for 1-2 minutes" -ForegroundColor Cyan

Start-Process $procmonPath -ArgumentList "/Quiet /Minimized /BackingFile $outputPML /LoadConfig C:\RemoteDiag\ps_filter.pmf" -PassThru

# Wait for designer to work
Write-Host "Capturing for 120 seconds..." -ForegroundColor Gray
Start-Sleep -Seconds 120

# Stop Process Monitor
Stop-Process -Name Procmon64 -Force

# Convert PML to CSV
Write-Host "Converting to CSV..." -ForegroundColor Yellow
Start-Process $procmonPath -ArgumentList "/OpenLog $outputPML /SaveAs $outputCSV" -Wait

Write-Host "✓ Process Monitor capture complete" -ForegroundColor Green
Write-Host "Output: $outputCSV"
```

### 2.3 Analyze Process Monitor Data (30 min)

```powershell
# Import and analyze CSV
$data = Import-Csv $outputCSV

# Find slowest operations
$slowOps = $data | Where-Object { [double]$_.Duration -gt 0.1 } | 
    Sort-Object { [double]$_.Duration } -Descending | 
    Select-Object -First 20

Write-Host "`nTop 20 Slowest Operations:" -ForegroundColor Cyan
$slowOps | Format-Table Time, Operation, Path, Duration -AutoSize

# Group by operation type
$opSummary = $data | Group-Object Operation | 
    Select-Object Name, Count | 
    Sort-Object Count -Descending

Write-Host "`nOperation Summary:" -ForegroundColor Cyan
$opSummary | Format-Table -AutoSize

# Find file access patterns
$fileOps = $data | Where-Object { $_.Operation -like "*File*" } |
    Group-Object Path | 
    Sort-Object Count -Descending | 
    Select-Object -First 10

Write-Host "`nMost Accessed Files:" -ForegroundColor Cyan
$fileOps | Format-Table Name, Count -AutoSize
```

### 2.4 Handle Analysis (Check for Locked Files) (30 min)

```powershell
# Run Handle utility
$handlePath = "C:\RemoteDiag\Tools\handle64.exe"
$handleOutput = & $handlePath -a Photoshop 2>&1

# Save output
$handleOutput | Out-File "C:\RemoteDiag\Output\handles.txt"

# Parse for file handles
$fileHandles = $handleOutput | Where-Object { $_ -match "File" }

Write-Host "`nPhotoshop File Handles:" -ForegroundColor Cyan
Write-Host "Total handles: $($fileHandles.Count)"

# Look for network paths
$networkHandles = $fileHandles | Where-Object { $_ -match "\\\\" }
if ($networkHandles) {
    Write-Host "`n⚠ Network file handles detected:" -ForegroundColor Yellow
    $networkHandles | ForEach-Object { Write-Host "  $_" }
}

# Look for temp files
$tempHandles = $fileHandles | Where-Object { $_ -match "Temp|tmp" }
if ($tempHandles) {
    Write-Host "`nTemp file handles: $($tempHandles.Count)" -ForegroundColor Gray
}
```

### 2.5 WPR Trace Capture (1 hour)

**Windows Performance Recorder can run via command line!**

```powershell
# Start WPR trace
Write-Host "Starting WPR trace..." -ForegroundColor Yellow
Write-Host "Tell designer to reproduce the slow operation NOW" -ForegroundColor Cyan

wpr -start CPU -start GPU -start DiskIO -filemode

Write-Host "`nTrace recording... work for 30-60 seconds" -ForegroundColor Green
Write-Host "Press Enter when designer completes the slow operation..."
Read-Host

# Stop and save
$etlFile = "C:\RemoteDiag\Output\photoshop_trace_$(Get-Date -Format 'yyyyMMdd_HHmmss').etl"
wpr -stop $etlFile

Write-Host "✓ Trace saved to: $etlFile" -ForegroundColor Green
Write-Host "File size: $([math]::Round((Get-Item $etlFile).Length / 1MB, 2)) MB"
```

**Note:** WPA (Windows Performance Analyzer) needs GUI. You have 2 options:

**Option A: Download trace file and analyze locally**
```bash
# From your local terminal
scp designer@192.168.1.XXX:C:/RemoteDiag/Output/photoshop_trace*.etl ./

# Open in WPA on your machine (if you have Windows)
wpa photoshop_trace_*.etl
```

**Option B: Use Remote Desktop for WPA analysis**
```powershell
# From SSH, enable Remote Desktop temporarily
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Then connect via RDP client from your machine
# IP: 192.168.1.XXX
# Credentials: designer's username/password
# Open WPA on their machine via RDP
```

### 2.6 Event Viewer Export (20 min)

```powershell
# Export recent Photoshop-related events
$eventFile = "C:\RemoteDiag\Output\events.csv"

$events = Get-WinEvent -FilterHashtable @{
    LogName = 'Application','System'
    Level = 1,2,3
    StartTime = (Get-Date).AddDays(-7)
} -ErrorAction SilentlyContinue | Where-Object {
    $_.Message -like "*Photoshop*" -or 
    $_.Message -like "*Adobe*" -or
    $_.ProviderName -like "*nvlddmkm*" -or
    $_.ProviderName -like "*amdwddmg*"
}

$events | Select-Object TimeCreated, LevelDisplayName, ProviderName, Id, Message |
    Export-Csv -Path $eventFile -NoTypeInformation

Write-Host "✓ Exported $($events.Count) events to $eventFile" -ForegroundColor Green

# Show critical errors
$critical = $events | Where-Object { $_.Level -eq 1 }
if ($critical) {
    Write-Host "`n⚠ $($critical.Count) Critical Errors:" -ForegroundColor Red
    $critical | Select-Object TimeCreated, Message | Format-List
}
```

---

## Day 3: GPU Diagnostics via SSH (3-4 hours)

### 3.1 GPU-Z Remote Monitoring (1 hour)

**GPU-Z has command-line logging mode!**

```powershell
$gpuzPath = "C:\RemoteDiag\Tools\GPU-Z.exe"
$logFile = "C:\RemoteDiag\Output\gpuz_log.txt"

# Start GPU-Z with logging
Write-Host "Starting GPU-Z logging..." -ForegroundColor Yellow
Start-Process $gpuzPath -ArgumentList "-log $logFile" -PassThru

Write-Host "GPU-Z is logging to: $logFile" -ForegroundColor Green
Write-Host "Let it run for 5 minutes while designer works..."
Write-Host "Press Enter to stop logging"
Read-Host

# Stop GPU-Z
Stop-Process -Name GPU-Z -Force

# Analyze log
$gpuData = Get-Content $logFile | Select-Object -Skip 1 | ConvertFrom-Csv

Write-Host "`nGPU Statistics:" -ForegroundColor Cyan
Write-Host "Samples collected: $($gpuData.Count)"

$avgGPULoad = ($gpuData.'GPU Load [%]' | Measure-Object -Average).Average
$maxGPULoad = ($gpuData.'GPU Load [%]' | Measure-Object -Maximum).Maximum
$avgTemp = ($gpuData.'GPU Temperature [°C]' | Measure-Object -Average).Average
$maxTemp = ($gpuData.'GPU Temperature [°C]' | Measure-Object -Maximum).Maximum

Write-Host "GPU Load:    Avg=$([math]::Round($avgGPULoad,1))%  Max=$([math]::Round($maxGPULoad,1))%"
Write-Host "Temperature: Avg=$([math]::Round($avgTemp,1))°C  Max=$([math]::Round($maxTemp,1))°C"
```

### 3.2 NVIDIA-SMI Monitoring (if NVIDIA GPU) (1 hour)

```powershell
# Check if nvidia-smi is available
$nvidiaSMI = Get-Command nvidia-smi -ErrorAction SilentlyContinue

if ($nvidiaSMI) {
    Write-Host "✓ NVIDIA GPU detected, starting nvidia-smi monitoring" -ForegroundColor Green
    
    # Create monitoring script
    $monitorScript = @'
$outputFile = "C:\RemoteDiag\Output\nvidia_monitor.csv"
"Timestamp,GPU_Util,Memory_Used,Memory_Total,Temp,Power,CoreClock,MemClock,ThrottleReasons" | Out-File $outputFile

for ($i = 0; $i -lt 300; $i++) {
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    
    $gpuUtil = nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits
    $memInfo = nvidia-smi --query-gpu=memory.used,memory.total --format=csv,noheader,nounits
    $temp = nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits
    $power = nvidia-smi --query-gpu=power.draw --format=csv,noheader,nounits
    $clocks = nvidia-smi --query-gpu=clocks.gr,clocks.mem --format=csv,noheader,nounits
    $throttle = nvidia-smi --query-gpu=clocks_throttle_reasons.active --format=csv,noheader
    
    $memUsed = $memInfo.Split(',')[0].Trim()
    $memTotal = $memInfo.Split(',')[1].Trim()
    $coreClock = $clocks.Split(',')[0].Trim()
    $memClock = $clocks.Split(',')[1].Trim()
    
    "$timestamp,$gpuUtil,$memUsed,$memTotal,$temp,$power,$coreClock,$memClock,$throttle" | Out-File $outputFile -Append
    
    if ($i % 10 -eq 0) {
        Write-Host "[$timestamp] GPU: $gpuUtil% | VRAM: $memUsed/$memTotal MB | Temp: ${temp}C"
    }
    
    Start-Sleep -Seconds 1
}
'@
    
    $monitorScript | Out-File "C:\RemoteDiag\Scripts\nvidia_monitor.ps1"
    
    # Run in background
    Write-Host "Starting 5-minute GPU monitoring..." -ForegroundColor Yellow
    Write-Host "Tell designer to work normally" -ForegroundColor Cyan
    
    Start-Job -FilePath "C:\RemoteDiag\Scripts\nvidia_monitor.ps1" -Name "GPUMonitor"
    
    # Wait for completion
    Wait-Job -Name "GPUMonitor" -Timeout 310
    
    # Get results
    $nvResults = Import-Csv "C:\RemoteDiag\Output\nvidia_monitor.csv"
    
    Write-Host "`nNVIDIA GPU Analysis:" -ForegroundColor Cyan
    
    $avgUtil = ($nvResults.GPU_Util | Measure-Object -Average).Average
    $avgVRAM = ($nvResults.Memory_Used | Measure-Object -Average).Average
    $maxVRAM = ($nvResults.Memory_Total | Select-Object -First 1)
    $vramPct = ($avgVRAM / $maxVRAM) * 100
    
    Write-Host "GPU Utilization: $([math]::Round($avgUtil,1))% average"
    Write-Host "VRAM Usage: $([math]::Round($vramPct,1))% average ($avgVRAM / $maxVRAM MB)"
    
    # Check for throttling
    $throttled = $nvResults | Where-Object { $_.ThrottleReasons -ne "0x0000000000000000" }
    if ($throttled) {
        Write-Host "⚠ Throttling detected in $($throttled.Count) samples" -ForegroundColor Yellow
    }
    
} else {
    Write-Host "⚠ nvidia-smi not available (AMD GPU or not in PATH)" -ForegroundColor Yellow
}
```

### 3.3 GPU Performance Counters (30 min)

```powershell
# Check available GPU counters
Write-Host "Available GPU Performance Counters:" -ForegroundColor Cyan
Get-Counter -ListSet GPU* -ErrorAction SilentlyContinue | Select-Object CounterSetName

# Monitor GPU Engine
$gpuCounters = Get-Counter "\GPU Engine(*)\Utilization Percentage" -Continuous -SampleInterval 1 -MaxSamples 180

Write-Host "`nMonitoring GPU for 3 minutes..." -ForegroundColor Yellow

$gpuSamples = @()
for ($i = 0; $i -lt 180; $i++) {
    $sample = Get-Counter "\GPU Engine(*engtype_3D)\Utilization Percentage" -ErrorAction SilentlyContinue
    
    if ($sample) {
        $gpuPct = [math]::Round(($sample.CounterSamples | Measure-Object CookedValue -Average).Average, 2)
        $gpuSamples += $gpuPct
        
        if ($i % 10 -eq 0) {
            Write-Host "  GPU: $gpuPct%"
        }
    }
    
    Start-Sleep -Seconds 1
}

$avgGPU = ($gpuSamples | Measure-Object -Average).Average
$maxGPU = ($gpuSamples | Measure-Object -Maximum).Maximum

Write-Host "`n✓ GPU Counter Analysis:" -ForegroundColor Green
Write-Host "Average: $([math]::Round($avgGPU,1))%"
Write-Host "Maximum: $([math]::Round($maxGPU,1))%"

if ($avgGPU -lt 30) {
    Write-Host "⚠ Low GPU usage - likely not being utilized by Photoshop" -ForegroundColor Yellow
}
```

---

## Day 4: Automated SSH Script Execution (3-4 hours)

### 4.1 Master Remote Diagnostic Script

**Create: `C:\RemoteDiag\Scripts\Remote_Master_Diag.ps1`**

```powershell
<#
.SYNOPSIS
    Master diagnostic script designed for SSH remote execution
.DESCRIPTION
    Runs all diagnostics automatically without GUI interaction
    Outputs to files for remote download and analysis
#>

$outputFolder = "C:\RemoteDiag\Output\Session_$(Get-Date -Format 'yyyyMMdd_HHmmss')"
New-Item -ItemType Directory -Path $outputFolder -Force | Out-Null

$reportFile = "$outputFolder\Report.txt"

function Write-Log {
    param($Message, $Color = "White")
    $timestamp = Get-Date -Format "HH:mm:ss"
    $logMessage = "[$timestamp] $Message"
    Write-Host $logMessage -ForegroundColor $Color
    $logMessage | Out-File -FilePath $reportFile -Append
}

Write-Log "=== REMOTE PHOTOSHOP DIAGNOSTICS ===" "Cyan"
Write-Log "Session: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"
Write-Log ""

# 1. System Information
Write-Log "[1/8] System Information" "Yellow"
$os = Get-CimInstance Win32_OperatingSystem
$cpu = Get-CimInstance Win32_Processor | Select-Object -First 1
$gpu = Get-CimInstance Win32_VideoController | Where-Object { $_.Name -notlike "*Microsoft*" }
$ram = [math]::Round($os.TotalVisibleMemorySize / 1MB, 2)

Write-Log "CPU: $($cpu.Name)"
Write-Log "RAM: $ram GB"
Write-Log "GPU: $($gpu.Name)"
Write-Log ""

# 2. Photoshop Status
Write-Log "[2/8] Photoshop Status" "Yellow"
$ps = Get-Process Photoshop -ErrorAction SilentlyContinue

if ($ps) {
    Write-Log "✓ Photoshop running (PID: $($ps.Id))" "Green"
    $psRAM = [math]::Round($ps.WorkingSet64 / 1GB, 2)
    Write-Log "  RAM: $psRAM GB"
    Write-Log "  Threads: $($ps.Threads.Count)"
} else {
    Write-Log "❌ Photoshop not running" "Red"
    Write-Log "Exiting - please start Photoshop first"
    exit 1
}
Write-Log ""

# 3. Performance Baseline
Write-Log "[3/8] Performance Baseline" "Yellow"
Start-Sleep -Seconds 2

$cpu = (Get-Counter "\Process(Photoshop)\% Processor Time" -ErrorAction SilentlyContinue).CounterSamples.CookedValue
$pf = (Get-Counter "\Process(Photoshop)\Page Faults/sec" -ErrorAction SilentlyContinue).CounterSamples.CookedValue

Write-Log "CPU: $([math]::Round($cpu,1))%"
Write-Log "Page Faults: $([math]::Round($pf,1))/sec"
Write-Log ""

# 4. Event Log Export
Write-Log "[4/8] Exporting Event Logs" "Yellow"
$eventFile = "$outputFolder\Events.csv"

$events = Get-WinEvent -FilterHashtable @{
    LogName = 'Application','System'
    Level = 1,2,3
    StartTime = (Get-Date).AddDays(-7)
} -ErrorAction SilentlyContinue | Where-Object {
    $_.Message -like "*Photoshop*" -or $_.Message -like "*Adobe*"
} | Select-Object -First 100

$events | Select-Object TimeCreated, LevelDisplayName, ProviderName, Message |
    Export-Csv -Path $eventFile -NoTypeInformation

Write-Log "✓ Exported $($events.Count) events"
Write-Log ""

# 5. Performance Monitoring
Write-Log "[5/8] Performance Monitoring (300 seconds)" "Yellow"
Write-Log "Designer should work normally now..."

$metricsFile = "$outputFolder\Metrics.csv"
"Timestamp,CPU_Pct,RAM_GB,PageFaults,DiskQueue,GPU_Pct" | Out-File $metricsFile

for ($i = 0; $i -lt 300; $i++) {
    $ps = Get-Process Photoshop -ErrorAction SilentlyContinue
    if (-not $ps) { break }
    
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $cpu = (Get-Counter "\Process(Photoshop)\% Processor Time" -ErrorAction SilentlyContinue).CounterSamples.CookedValue
    $ram = [math]::Round($ps.WorkingSet64 / 1GB, 2)
    $pf = (Get-Counter "\Process(Photoshop)\Page Faults/sec" -ErrorAction SilentlyContinue).CounterSamples.CookedValue
    $diskQ = (Get-Counter "\PhysicalDisk(_Total)\Current Disk Queue Length" -ErrorAction SilentlyContinue).CounterSamples.CookedValue
    
    $gpuPct = 0
    try {
        $gpu = Get-Counter "\GPU Engine(*engtype_3D)\Utilization Percentage" -ErrorAction Stop
        $gpuPct = [math]::Round(($gpu.CounterSamples | Measure-Object CookedValue -Average).Average, 2)
    } catch {}
    
    "$timestamp,$([math]::Round($cpu,2)),$ram,$([math]::Round($pf,2)),$([math]::Round($diskQ,2)),$gpuPct" | 
        Out-File $metricsFile -Append
    
    if ($i % 30 -eq 0) {
        Write-Log "  [$i/300] CPU:$([math]::Round($cpu,1))% RAM:${ram}GB GPU:$gpuPct%"
    }
    
    Start-Sleep -Seconds 1
}

Write-Log "✓ Monitoring complete"
Write-Log ""

# 6. Analysis
Write-Log "[6/8] Analyzing Data" "Yellow"
$data = Import-Csv $metricsFile

$avgCPU = ($data.CPU_Pct | Measure-Object -Average).Average
$avgRAM = ($data.RAM_GB | Measure-Object -Average).Average
$avgPF = ($data.PageFaults | Measure-Object -Average).Average
$avgGPU = ($data.GPU_Pct | Measure-Object -Average).Average

Write-Log "CPU Average: $([math]::Round($avgCPU,1))%"
Write-Log "RAM Average: $([math]::Round($avgRAM,1)) GB"
Write-Log "Page Faults: $([math]::Round($avgPF,1))/sec"
Write-Log "GPU Average: $([math]::Round($avgGPU,1))%"
Write-Log ""

# 7. Recommendations
Write-Log "[7/8] Recommendations" "Yellow"

if ($avgCPU -gt 85) {
    Write-Log "🔴 CPU BOTTLENECK: $([math]::Round($avgCPU,1))% average" "Red"
}
if ($avgPF -gt 100) {
    Write-Log "🔴 RAM SHORTAGE: $([math]::Round($avgPF,1)) page faults/sec" "Red"
}
if ($avgGPU -lt 30) {
    Write-Log "🟡 LOW GPU USAGE: $([math]::Round($avgGPU,1))% average" "Yellow"
}
if ($avgCPU -lt 70 -and $avgPF -lt 50 -and $avgGPU -gt 50) {
    Write-Log "✅ System performing well" "Green"
}

Write-Log ""

# 8. File Summary
Write-Log "[8/8] Session Complete" "Green"
Write-Log ""
Write-Log "Output files in: $outputFolder"
Write-Log "- Report.txt (this file)"
Write-Log "- Metrics.csv (performance data)"
Write-Log "- Events.csv (system events)"
Write-Log ""
Write-Log "Download files using: scp user@host:$outputFolder/* ./"

# Compress output for easy download
Write-Log "Creating ZIP archive..." "Yellow"
$zipFile = "$outputFolder.zip"
Compress-Archive -Path $outputFolder -DestinationPath $zipFile -Force
Write-Log "✓ Archive: $zipFile"
```

### 4.2 Run Master Script via SSH

```bash
# From SSH session
ssh designer@192.168.1.XXX

# Switch to PowerShell
powershell.exe

# Run master diagnostic
cd C:\RemoteDiag\Scripts
.\Remote_Master_Diag.ps1

# Script runs for ~6-7 minutes total
# Outputs everything to timestamped folder
```

### 4.3 Download Results

```bash
# From your local machine
# Download the ZIP file
scp designer@192.168.1.XXX:C:/RemoteDiag/Output/Session_*.zip ./

# Extract and analyze locally
unzip Session_*.zip

# Review Report.txt
cat Session_*/Report.txt

# Import Metrics.csv into Excel for charts
```

---

## Day 5: Continuous Monitoring Setup (2-3 hours)

### 5.1 Background Monitoring Service

**Create monitoring script that runs continuously:**

```powershell
# Create: C:\RemoteDiag\Scripts\ContinuousMonitor.ps1

$logFile = "C:\RemoteDiag\Output\continuous_$(Get-Date -Format 'yyyyMMdd').csv"

# Create CSV header if new file
if (-not (Test-Path $logFile)) {
    "Timestamp,CPU_Pct,RAM_GB,PageFaults_Sec,GPU_Pct,DiskQueue,Status" | Out-File $logFile
}

while ($true) {
    $ps = Get-Process Photoshop -ErrorAction SilentlyContinue
    
    if ($ps) {
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $cpu = (Get-Counter "\Process(Photoshop)\% Processor Time" -EA SilentlyContinue).CounterSamples.CookedValue
        $ram = [math]::Round($ps.WorkingSet64 / 1GB, 2)
        $pf = (Get-Counter "\Process(Photoshop)\Page Faults/sec" -EA SilentlyContinue).CounterSamples.CookedValue
        $diskQ = (Get-Counter "\PhysicalDisk(_Total)\Current Disk Queue Length" -EA SilentlyContinue).CounterSamples.CookedValue
        
        $gpuPct = 0
        try {
            $gpu = Get-Counter "\GPU Engine(*engtype_3D)\Utilization Percentage" -EA Stop
            $gpuPct = [math]::Round(($gpu.CounterSamples | Measure-Object CookedValue -Average).Average, 2)
        } catch {}
        
        # Determine status
        $status = "OK"
        if ($cpu -gt 90) { $status = "HIGH_CPU" }
        if ($pf -gt 100) { $status = "HIGH_PAGE_FAULTS" }
        if ($diskQ -gt 5) { $status = "DISK_BOTTLENECK" }
        
        "$timestamp,$([math]::Round($cpu,2)),$ram,$([math]::Round($pf,2)),$gpuPct,$([math]::Round($diskQ,2)),$status" | 
            Out-File $logFile -Append
    }
    
    Start-Sleep -Seconds 5
}
```

### 5.2 Start Background Monitor via SSH

```powershell
# From SSH session
# Start monitor in background job
Start-Job -FilePath "C:\RemoteDiag\Scripts\ContinuousMonitor.ps1" -Name "PSMonitor"

# Verify it's running
Get-Job

# Job will run continuously, logging every 5 seconds
# Creates daily log files automatically
```

### 5.3 Check Monitoring Status Remotely

```powershell
# From SSH, check if monitor is still running
Get-Job -Name "PSMonitor"

# View recent data
$todayLog = "C:\RemoteDiag\Output\continuous_$(Get-Date -Format 'yyyyMMdd').csv"
Get-Content $todayLog -Tail 20

# Quick analysis of today's data
$data = Import-Csv $todayLog
$issues = $data | Where-Object { $_.Status -ne "OK" }

Write-Host "Today's monitoring:"
Write-Host "Total samples: $($data.Count)"
Write-Host "Issues detected: $($issues.Count)"

if ($issues) {
    Write-Host "`nIssue breakdown:"
    $issues | Group-Object Status | Select-Object Name, Count | Format-Table
}
```

### 5.4 Stop Background Monitor

```powershell
# When done
Stop-Job -Name "PSMonitor"
Remove-Job -Name "PSMonitor"
```

---

## Day 6: Final Report & Cleanup (2-3 hours)

### 6.1 Generate Comprehensive Report

```powershell
# From SSH
cd C:\RemoteDiag\Scripts

# Run report generator
.\Generate_Report.ps1 -DiagFolder "C:\RemoteDiag\Output\Session_*"

# Downloads HTML report automatically
# Open in browser to review
```

### 6.2 Download All Diagnostic Data

```bash
# From your local machine
# Download entire output folder
scp -r designer@192.168.1.XXX:C:/RemoteDiag/Output ./Photoshop_Diagnostics/

# Download specific files
scp designer@192.168.1.XXX:C:/RemoteDiag/Output/Session_*.zip ./
scp designer@192.168.1.XXX:C:/RemoteDiag/Output/*.etl ./  # WPR traces
```

### 6.3 Cleanup (Optional)

```powershell
# From SSH, cleanup diagnostic files if needed
Remove-Item "C:\RemoteDiag\Output\*" -Recurse -Force

# Keep scripts for future use
# Remove tools if not needed
Remove-Item "C:\RemoteDiag\Tools\*" -Force
```

---

## SSH Session Management Tips

### Keep SSH Session Alive

```bash
# In your SSH config (~/.ssh/config on Linux/Mac)
Host photoshop-diag
    HostName 192.168.1.XXX
    User designer
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Then connect with:
ssh photoshop-diag
```

### Run Long Tasks Without Disconnection

```powershell
# Start a task that continues even if SSH disconnects
Start-Job -ScriptBlock {
    # Your long-running command
    C:\RemoteDiag\Scripts\Remote_Master_Diag.ps1
} -Name "LongTask"

# Disconnect SSH safely
# Reconnect later and check:
Get-Job -Name "LongTask"
Receive-Job -Name "LongTask"
```

### Multiple SSH Sessions

```bash
# Terminal 1: Run diagnostics
ssh designer@192.168.1.XXX
cd C:\RemoteDiag
.\Scripts\Remote_Master_Diag.ps1

# Terminal 2: Monitor in real-time
ssh designer@192.168.1.XXX
Get-Content C:\RemoteDiag\Output\Session_*\Report.txt -Wait

# Terminal 3: Download results as they're created
scp designer@192.168.1.XXX:C:/RemoteDiag/Output/Session_*/Metrics.csv ./
```

---

## Complete SSH Workflow Summary

### What You CAN Do via SSH:
✅ All PowerShell scripts
✅ Performance Monitor data collection
✅ Process Monitor (command-line mode)
✅ Event Viewer exports
✅ GPU-Z logging
✅ nvidia-smi monitoring
✅ WPR trace capture
✅ Handle analysis
✅ Registry exports
✅ System information gathering
✅ Real-time performance monitoring
✅ File downloads (scp)
✅ Background job execution

### What Requires GUI/RDP:
❌ WPA analysis (Windows Performance Analyzer)
❌ Process Explorer GUI features
❌ GPU-Z visual interface
❌ Opening Photoshop documents
❌ Testing Photoshop operations manually

### Workaround for GUI Tools:
- **Option 1:** Enable Remote Desktop temporarily, analyze via RDP
- **Option 2:** Download files, analyze on your local Windows machine
- **Option 3:** Designer shares screen via TeamViewer/AnyDesk while you run commands via SSH

---

## Final Budget & Timeline

### SSH-Based Diagnostics:

**Minimum (Basic Remote Analysis):**
- **4 days, 15 hours total**
- **Budget: $45 USD / ₹3,780 INR**
- SSH setup + automated scripts + basic analysis

**Recommended (Comprehensive Remote Diagnostics):**
- **5 days, 20 hours total**
- **Budget: $60 USD / ₹5,040 INR**
- Full tool suite + continuous monitoring + detailed reports

**Maximum (Deep Investigation + GUI Analysis):**
- **6 days, 24 hours total**
- **Budget: $72 USD / ₹6,048 INR**
- Everything above + RDP sessions for WPA/GUI tools

### Advantages of SSH Approach:
- Designer can work uninterrupted
- No need to be on-site
- Can monitor 24/7 with background jobs
- Easy to re-run tests
- All data automatically saved for analysis

### Deliverables via SSH:
- Complete diagnostic reports (TXT + HTML)
- Performance metrics (CSV files)
- WPR trace files (for local analysis)
- Event logs export
- GPU monitoring data
- Continuous monitoring logs
- All scripts for future use

**This approach works perfectly for remote diagnostics while designer continues working!**
