        # Complete SSH Remote Diagnostics - Organized by Category

**Project Complexity: High**
**Budget: $45-72 USD / ₹3,780-6,048 INR**
**Timeline: 5-6 days (15-24 hours total)**

---

## What We'll Diagnose via SSH

### 1. CPU Performance Analysis

**What We Test:**
- Real-time CPU usage % during Photoshop work
- Per-core CPU utilization (is one core maxed out?)
- CPU usage patterns during filters, brushes, saves
- Thread count and efficiency
- Context switches (thread contention)

**What We Identify:**
- ❌ Single-core bottleneck (one core at 100%, others idle)
- ❌ Multi-core saturation (all cores maxed)
- ❌ CPU not powerful enough for workload
- ✅ CPU performing normally

**SSH Commands Used:**
```powershell
# Real-time CPU monitoring
Get-Counter "\Process(Photoshop)\% Processor Time"
Get-Counter "\Processor(*)\% Processor Time"  # Per-core

# Thread analysis
(Get-Process Photoshop).Threads.Count
Get-Counter "\Thread(*photoshop*)\Context Switches/sec"
```

**Tells You:**
- "Your CPU is bottlenecked - only 1 core at 100%, others at 20%"
- "All 8 cores at 95% - CPU upgrade needed"
- OR "CPU is fine at 40% usage, problem is elsewhere"

---

### 2. RAM/Memory Issues

**What We Test:**
- Photoshop RAM usage (current GB used)
- Page faults per second (disk access due to RAM shortage)
- RAM allocation vs available
- Memory pressure on system
- Scratch disk usage patterns

**What We Identify:**
- ❌ RAM shortage (high page faults >100/sec)
- ❌ RAM not allocated properly in Photoshop preferences
- ❌ Memory leak (RAM usage grows without reason)
- ❌ Too many programs competing for RAM
- ✅ RAM is sufficient

**SSH Commands Used:**
```powershell
# RAM monitoring
$ps = Get-Process Photoshop
[math]::Round($ps.WorkingSet64/1GB, 2)  # Current RAM

# Page faults (critical metric)
Get-Counter "\Process(Photoshop)\Page Faults/sec"
Get-Counter "\Memory\Page Reads/sec"
```

**Tells You:**
- "32GB RAM but Photoshop only using 8GB - change settings to 75%"
- "Page faults at 250/sec - RAM shortage, add more RAM"
- OR "RAM usage healthy at 16GB with no paging"

---

### 3. GPU Problems

**What We Test:**
- GPU utilization % during Photoshop work
- VRAM (video memory) usage
- GPU temperature and throttling
- GPU clock speeds
- GPU driver version and age
- GPU acceleration enabled/disabled in Photoshop

**What We Identify:**
- ❌ GPU not being used (<30% usage)
- ❌ GPU disabled in Photoshop preferences
- ❌ Outdated GPU drivers
- ❌ GPU overheating and throttling
- ❌ VRAM maxed out (>90%)
- ❌ GPU hardware failure/issues
- ✅ GPU working properly

**SSH Commands Used:**
```powershell
# GPU monitoring
Get-Counter "\GPU Engine(*engtype_3D)\Utilization Percentage"

# NVIDIA specific
nvidia-smi --query-gpu=utilization.gpu,memory.used,temperature.gpu --format=csv

# GPU-Z logging (command-line mode)
Start-Process GPU-Z.exe -ArgumentList "-log C:\gpuz.txt"
```

**Tells You:**
- "GPU at 15% - Photoshop not using it, enable in preferences"
- "GPU driver 2 years old - update may fix issues"
- "GPU at 88°C causing throttling - clean dust"
- "VRAM 95% full - document too large for 4GB GPU"
- OR "GPU at 75%, performing well"

---

### 4. Disk/Storage Bottlenecks

**What We Test:**
- Disk queue length (how many operations waiting)
- Disk active time %
- Read/write speeds
- Scratch disk location and speed
- File access patterns
- Disk fragmentation (HDD only)

**What We Identify:**
- ❌ Slow HDD as scratch disk
- ❌ Scratch disk full (<20GB free)
- ❌ Disk queue constantly >5 (bottleneck)
- ❌ NVMe running at SATA speeds (misconfigured)
- ❌ Network drive causing delays
- ✅ Storage performing well

**SSH Commands Used:**
```powershell
# Disk monitoring
Get-Counter "\PhysicalDisk(_Total)\Current Disk Queue Length"
Get-Counter "\PhysicalDisk(*)\% Disk Time"

# Process Monitor (captures file operations)
Start-Process Procmon64.exe -ArgumentList "/Quiet /BackingFile capture.pml"
```

**Tells You:**
- "Scratch disk is slow 7200RPM HDD - upgrade to NVMe"
- "Disk queue at 12 - disk can't keep up"
- "Only 5GB free on scratch disk - free up space"
- "Working from \\server causing 2-second delays"
- OR "NVMe SSD performing perfectly at 3500 MB/s"

---

### 5. Photoshop Software Issues

**What We Test:**
- Photoshop version and updates
- Preferences file corruption
- Plugin conflicts
- GPU settings in Photoshop
- Scratch disk configuration
- History states and cache settings
- Extensions and CEF Helper resource usage

**What We Identify:**
- ❌ Corrupted preferences file
- ❌ Old plugin causing slowdowns
- ❌ GPU disabled when it should be enabled
- ❌ Wrong scratch disk selected
- ❌ Too many history states (>50)
- ❌ Extension/panel consuming resources
- ✅ Photoshop configured correctly

**SSH Commands Used:**
```powershell
# Check preferences file
$prefsPath = "$env:APPDATA\Adobe\Adobe Photoshop 2024\*.psp"
Get-Item $prefsPath | Select-Object Length, LastWriteTime

# List plugins
Get-ChildItem "C:\Program Files\Adobe\*\Plug-ins" -Recurse

# Check CEF Helper resource usage
Get-Process "Adobe CEF Helper" | Select-Object CPU, WorkingSet64
```

**Tells You:**
- "Preferences file 45MB (should be <5MB) - corrupted, reset needed"
- "Plugin 'OldFilter.8bf' from 2018 taking 5s to load"
- "GPU disabled in preferences - enable for 3x faster filters"
- "Scratch disk on C: with only 10GB free - change to D: (500GB free)"
- OR "Photoshop configured optimally"

---

### 6. Driver & System Issues

**What We Test:**
- GPU driver version and crashes
- DPC/ISR latency (driver interference)
- Event log errors (crashes, warnings)
- System file integrity
- Reliability history (crash patterns)
- Windows updates status

**What We Identify:**
- ❌ GPU driver crashes/resets
- ❌ High DPC latency from bad driver (mouse lag, stuttering)
- ❌ Photoshop crash history
- ❌ System file corruption
- ❌ Driver conflicts
- ✅ Drivers stable and updated

**SSH Commands Used:**
```powershell
# Event log export
Get-WinEvent -FilterHashtable @{
    LogName = 'Application','System'
    Level = 1,2,3
    StartTime = (Get-Date).AddDays(-7)
} | Where-Object { $_.Message -like "*Photoshop*" }

# Check driver versions
$gpu = Get-CimInstance Win32_VideoController
$gpu.DriverVersion
$gpu.DriverDate

# System integrity
sfc /scannow
DISM /Online /Cleanup-Image /CheckHealth
```

**Tells You:**
- "GPU driver crashed 3 times this week - Event ID 4101"
- "WiFi driver causing 2000µs DPC latency - update or disable"
- "Photoshop crashed 8 times in 7 days - GPU-related pattern"
- "System files corrupted - DISM repair needed"
- OR "All drivers current, no crashes detected"

---

### 7. Network/Cloud Issues

**What We Test:**
- Network file access (UNC paths)
- Cloud sync activity during work
- Network latency to file locations
- Open file handles to network shares

**What We Identify:**
- ❌ Working from network drive causing delays
- ❌ Creative Cloud sync during work
- ❌ Network disconnections
- ❌ Slow network storage
- ✅ No network interference

**SSH Commands Used:**
```powershell
# Check for network file handles
& handle64.exe -a Photoshop | Where-Object { $_ -match "\\\\" }

# Process Monitor network operations
# Filter: Operation = Network Read/Write

# Check network paths in use
Get-SmbMapping
```

**Tells You:**
- "Opening files from \\server\projects taking 5s each - copy locally"
- "Creative Cloud syncing during work causing 100MB/s upload - pause sync"
- OR "All files local, no network delays"

---

### 8. Specific Photoshop Operation Performance

**What We Test:**
- Filter application time (Gaussian Blur benchmark)
- Brush responsiveness and lag
- Document open/save time
- Layer operations speed
- Undo/Redo performance

**What We Identify:**
- ❌ Specific filters taking too long
- ❌ Brush lag/delay
- ❌ Slow saves (5+ minutes)
- ❌ Slow document loading
- ✅ All operations within expected time

**SSH Commands Used:**
```powershell
# WPR trace during operation
wpr -start CPU -start GPU -filemode
# (User applies filter)
wpr -stop filter_trace.etl

# Monitor during specific operation
while ($true) {
    Get-Counter "\Process(Photoshop)\% Processor Time"
    Get-Counter "\GPU Engine(*)\Utilization Percentage"
    Start-Sleep -Seconds 1
}
```

**Tells You:**
- "Gaussian Blur 500px taking 45s (should be 5-10s) - GPU issue"
- "Brush has 200ms lag - CPU bottleneck or GPU disabled"
- "Saving 500MB file takes 8 minutes - disk bottleneck"
- OR "All operations performing at expected speeds"

---

## Complete Diagnostic Workflow via SSH

### Day 1: System Overview (3-4 hours)
- Connect via SSH
- Check Photoshop status
- System information gathering
- Start Performance Monitor collection
- Real-time monitoring while designer works

### Day 2: Deep Analysis (4-5 hours)
- Download/install diagnostic tools remotely
- Process Monitor capture
- Handle analysis
- WPR trace capture
- Event Viewer export

### Day 3: GPU Focus (3-4 hours)
- GPU-Z logging
- NVIDIA-SMI monitoring (if NVIDIA)
- GPU performance counters
- Temperature and throttling analysis

### Day 4: Automated Scripts (3-4 hours)
- Run master diagnostic script
- Collects all metrics automatically
- Creates comprehensive reports
- Downloads ZIP file with all results

### Day 5: Continuous Monitoring (2-3 hours)
- Setup background monitoring
- 24/7 logging to CSV files
- Alert on performance issues
- Daily summary reports

### Day 6: Final Report (2-3 hours)
- Generate HTML report with charts
- Download all diagnostic data
- Present findings to client
- Cleanup (optional)

---

## What You Get (Deliverables)

### 1. Real-Time Dashboard
```
[10:23:15] CPU: 87% | RAM: 18.5GB | PageFaults: 145/s | GPU: 23% | DiskQ: 8
[10:23:25] CPU: 92% | RAM: 18.7GB | PageFaults: 152/s | GPU: 19% | DiskQ: 9
```

### 2. Performance Metrics CSV
Import into Excel → Create graphs showing trends

### 3. Text Report
```
=== DIAGNOSIS ===
🔴 CPU BOTTLENECK: 87% average
🔴 HIGH PAGE FAULTS: 145/sec (RAM shortage)
🟡 LOW GPU: 23% (underutilized)
🟡 DISK QUEUE: 8.2 sustained

=== RECOMMENDATIONS ===
1. [FREE] Increase RAM to 75% in Photoshop
2. [FREE] Update GPU driver
3. [FREE] Change scratch disk to NVMe
4. [$400] CPU upgrade (if needed)
```

### 4. HTML Visual Report
Opens in browser with:
- Line graphs (CPU/RAM/GPU over time)
- Bottleneck summary with color coding
- Timeline of events
- Prioritized recommendations

### 5. Supporting Data Files
- Event logs (CSV)
- GPU monitoring logs
- Process Monitor captures
- WPR traces (for advanced analysis)
- Plugin analysis
- Registry exports

---

## Final Budget Options

### Option 1: Basic Analysis
**$45 USD / ₹3,780 INR**
- 15 hours over 4 days
- Core diagnostics + automated scripts
- Identifies main bottleneck
- Basic recommendations

### Option 2: Comprehensive (Recommended)
**$60 USD / ₹5,040 INR**
- 20 hours over 5 days
- All diagnostics + GPU deep-dive
- Continuous monitoring setup
- Detailed reports + graphs
- Before/after comparisons

### Option 3: Maximum Investigation
**$72 USD / ₹6,048 INR**
- 24 hours over 6 days
- Everything in Option 2
- RDP sessions for GUI tools (WPA)
- Multiple test iterations
- Custom script development

---

## Key Advantages

✅ **Zero disruption** - Designer works normally
✅ **Remote** - No need to be on-site
✅ **24/7 monitoring** - Background jobs continue
✅ **Repeatable** - Easy to re-run tests
✅ **Automated** - Scripts do heavy lifting
✅ **Complete data** - Everything saved for analysis
✅ **Professional reports** - Client-ready deliverables

**All via SSH while designer uses Photoshop!**
