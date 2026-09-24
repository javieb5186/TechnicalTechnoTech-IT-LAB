# Automated Server Health Monitoring & Service Accounts

## Project Overview

This project explores how **Windows service identities and domain service accounts** can be used to run automated administrative workloads.

I created a **PowerShell server health monitoring script**, automated it using **Windows Task Scheduler**, and configured the task to run independently of my administrator account.

The project then progressed from a built-in **Local Service account** toward a dedicated **Active Directory domain service account** with access to a centralized SMB share.

The domain service account configuration encountered a logon issue and was intentionally left unresolved for future troubleshooting.

***

## Lab Environment

**Domain:** `technicaltechnotech.com`  
**NetBIOS Domain Name:** `TTT`

### Systems

- **DC01** — Windows Server Domain Controller
- **DC02** — Windows Server Core Domain Controller
- **MGMT01** — Windows management workstation

### Technologies Used

- Active Directory Domain Services
- Windows PowerShell
- PowerShell ISE
- Windows Task Scheduler
- Windows Server Core
- SMB File Sharing
- NTFS Permissions
- PowerShell Remoting

***

## Project Architecture

The health-monitoring workload was configured on **MGMT01**.

```text
MGMT01
│
├── PowerShell Health Monitoring Script
│
└── Windows Task Scheduler
        │
        ├── Stage 1: LOCAL SERVICE
        │
        └── Stage 2: TTT\svc_ServerHealth
                            │
                            │ SMB
                            ▼
                 \\DC02\ServerHealthLogs
```

The long-term goal was to progress through three service identity models:

```text
Local Service
     ↓
Traditional Domain Service Account
     ↓
Group Managed Service Account (gMSA)
```

The project was stopped during the traditional domain service account stage.

***

# Stage 1 — Create the Server Health Monitoring Script

I created the following directory on **MGMT01**:

```text
C:\TTT\ServerHealth
```

The PowerShell script was saved as:

```text
C:\TTT\ServerHealth\ServerHealth.ps1
```

## PowerShell Script

```powershell
$LogFile = "C:\TTT\ServerHealth\ServerHealth.log"

$Time = Get-Date
$OS = Get-CimInstance Win32_OperatingSystem

$Uptime = $Time - $OS.LastBootUpTime

$Disk = Get-CimInstance Win32_LogicalDisk -Filter "DeviceID='C:'"
$FreeGB = [math]::Round($Disk.FreeSpace / 1GB, 2)

$MemoryUsed = [math]::Round(
    ($OS.TotalVisibleMemorySize - $OS.FreePhysicalMemory) / 1MB,
    2
)

$Report = @"

==============================
Server Health Report
==============================
Computer: $env:COMPUTERNAME
Time: $Time
Uptime: $($Uptime.Days) days, $($Uptime.Hours) hours
C: Free Space: $FreeGB GB
Memory Used: $MemoryUsed GB

"@

Add-Content -Path $LogFile -Value $Report
```

### Health Information Collected

The script collected:

- Computer name
- Current date and time
- System uptime
- Available disk space
- Memory usage

The script then wrote the information to a log file.

***

## Manual Testing

I manually executed:

```powershell
C:\TTT\ServerHealth\ServerHealth.ps1
```

The script successfully generated:

```text
C:\TTT\ServerHealth\ServerHealth.log
```

This confirmed that the PowerShell monitoring script worked before automation was introduced.

***

# Stage 2 — Automate the Script with Task Scheduler

I created a scheduled task on **MGMT01** called:

```text
TTT Server Health Monitor
```

The task executed:

```text
powershell.exe
```

with the following arguments:

```text
-NoProfile -ExecutionPolicy Bypass -File "C:\TTT\ServerHealth\ServerHealth.ps1"
```

***

# Stage 3 — Run the Task as Local Service

Instead of running the task using my administrator account, I configured it to use:

```text
NT AUTHORITY\LOCAL SERVICE
```

The task successfully executed the PowerShell script and generated the local health report.

### Local Service Workflow

```text
Task Scheduler
      │
      ▼
NT AUTHORITY\LOCAL SERVICE
      │
      ▼
ServerHealth.ps1
      │
      ▼
C:\TTT\ServerHealth\ServerHealth.log
```

**Result:** Successful

This demonstrated that an automated workload does not necessarily need to run using a normal user or administrator account.

***

# Stage 4 — Create a Domain Service Account

The next objective was to allow the monitoring workload to access resources on another domain computer.

I created a dedicated Organizational Unit:

```text
Service Accounts
```

Inside the OU, I created:

```text
TTT\svc_ServerHealth
```

The account was designed specifically for the monitoring workload rather than being used interactively by an administrator.

The service account was **not** added to privileged groups such as:

- Domain Admins
- Enterprise Admins
- Administrators

This followed the **principle of least privilege**.

***

# Stage 5 — Create a Centralized SMB Share

Instead of keeping health reports only on MGMT01, I attempted to centralize them on **DC02**.

The target was:

```text
\\DC02\ServerHealthLogs
```

Because DC02 was running **Windows Server Core**, I configured the folder remotely using PowerShell from MGMT01.

## Create the Directory Remotely

```powershell
Invoke-Command -ComputerName DC02 -ScriptBlock {
    New-Item -Path "C:\ServerHealthLogs" -ItemType Directory
}
```

## Create the SMB Share

```powershell
Invoke-Command -ComputerName DC02 -ScriptBlock {
    New-SmbShare `
        -Name "ServerHealthLogs" `
        -Path "C:\ServerHealthLogs" `
        -ChangeAccess "TTT\svc_ServerHealth"
}
```

The resulting UNC path was:

```text
\\DC02\ServerHealthLogs
```

***

# Stage 6 — Configure Share and NTFS Permissions

Two permission layers were required:

```text
TTT\svc_ServerHealth
        │
        ▼
SMB Share Permission
      Change
        │
        ▼
NTFS Permission
      Modify
        │
        ▼
C:\ServerHealthLogs
```

## Share Permission

The service account was granted:

```text
Change
```

This was verified remotely using:

```powershell
Get-SmbShareAccess -CimSession DC02 -Name ServerHealthLogs
```

## NTFS Permission

The service account was granted:

```text
Modify, Synchronize
```

This reinforced an important SMB concept:

> **Accessing a Windows file share can involve both Share permissions and NTFS permissions.**

***

# Stage 7 — Test Local Service Against the Network Resource

The monitoring script was changed from a local destination:

```powershell
$LogFile = "C:\TTT\ServerHealth\ServerHealth.log"
```

to the centralized SMB location:

```powershell
$LogFile = "\\DC02\ServerHealthLogs\MGMT01-ServerHealth.log"
```

The scheduled task was still running as:

```text
NT AUTHORITY\LOCAL SERVICE
```

When the scheduled task attempted to write to the protected network share, access was denied.

### Test Result

```text
MGMT01
   │
   ▼
LOCAL SERVICE
   │
   │ SMB request
   ▼
DC02
   │
   ▼
ServerHealthLogs
   │
   └── ACCESS DENIED
```

This demonstrated why a workload that needs access to protected domain resources may require an appropriate **domain identity**.

***

# Stage 8 — Configure the Domain Service Account

I changed the scheduled task identity from:

```text
NT AUTHORITY\LOCAL SERVICE
```

to:

```text
TTT\svc_ServerHealth
```

Windows Task Scheduler generated a warning involving:

```text
Log on as a batch job
```

## Log on as a Batch Job

Because the workload was being executed through **Task Scheduler**, the domain service account required:

```text
Log on as a batch job
```

This was configured through:

```text
Local Security Policy
    ↓
Local Policies
    ↓
User Rights Assignment
    ↓
Log on as a batch job
```

The account added was:

```text
TTT\svc_ServerHealth
```

This demonstrated an important distinction:

| Workload | Required User Right |
|---|---|
| Scheduled Task | **Log on as a batch job** |
| Windows Service | **Log on as a service** |

***

# Troubleshooting

After configuring the batch logon right, the scheduled task still failed.

I investigated the issue using:

```text
Event Viewer
→ Applications and Services Logs
→ Microsoft
→ Windows
→ TaskScheduler
→ Operational
```

Relevant events included:

- **Event ID 101**
- **Event ID 104**

The detailed event indicated that Task Scheduler failed during the account logon process.

The error referenced:

```text
LogonUserExEx
```

and indicated that the account credentials/logon configuration should be checked.

## Troubleshooting Boundary

The troubleshooting process helped identify exactly where the failure occurred:

```text
Scheduled Task
      │
      ▼
Log on TTT\svc_ServerHealth
      │
      ❌ Logon failure
      │
      X
PowerShell Script
      │
      X
SMB Authentication
      │
      X
\\DC02\ServerHealthLogs
```

The final failure occurred **before the script could test access to the SMB share**.

The SMB and NTFS permissions had already been configured and verified, but the service account could not successfully establish the required Task Scheduler logon session.

Rather than continuing to troubleshoot the account logon configuration, I documented the issue for future investigation.

***

# Final Project Status

| Component | Status |
|---|---|
| PowerShell health monitoring script | Working |
| Local health report | Working |
| Task Scheduler automation | Working |
| Local Service execution | Working |
| Service Accounts OU | Created |
| Domain service account | Created |
| Remote Server Core administration | Working |
| SMB share on DC02 | Created |
| Share permission | Change |
| NTFS permission | Modify |
| Local Service network test | Access denied as expected |
| Log on as a batch job | Configured |
| Domain service account scheduled task | **Unresolved logon failure** |
| gMSA migration | Not attempted |

***

# What I Learned

- Created a PowerShell-based server health monitoring script
- Used PowerShell to collect operating system, uptime, disk, and memory information
- Automated a PowerShell workload using Windows Task Scheduler
- Ran a scheduled workload using the built-in **Local Service** identity
- Created a dedicated Active Directory service account
- Created a **Service Accounts OU**
- Applied the **principle of least privilege**
- Remotely administered Windows Server Core using PowerShell
- Created an SMB share remotely
- Configured SMB Share permissions
- Configured NTFS permissions
- Learned the difference between **Share permissions and NTFS permissions**
- Learned why domain identities are useful for workloads accessing protected network resources
- Learned the difference between **Log on as a batch job** and **Log on as a service**
- Used Event Viewer to troubleshoot Task Scheduler
- Identified a Task Scheduler failure occurring during the account logon process
- Practiced identifying exactly where in a workflow a failure occurs

***

# Skills Practiced

- Active Directory Domain Services (AD DS)
- Service Accounts
- Windows PowerShell
- PowerShell ISE
- PowerShell Remoting
- Windows Task Scheduler
- Windows Server Core
- SMB File Sharing
- NTFS Permissions
- Share Permissions
- Principle of Least Privilege
- Local Service
- Domain Service Accounts
- User Rights Assignment
- Log on as a Batch Job
- Event Viewer
- Task Scheduler Troubleshooting
- `Get-CimInstance`
- `Invoke-Command`
- `New-SmbShare`
- `Get-SmbShareAccess`
- `Get-Acl`
- `Set-Acl`

***

# Future Improvements

This project can be revisited later to:

1. Resolve the `TTT\svc_ServerHealth` Task Scheduler logon issue.
2. Confirm authenticated access to `\\DC02\ServerHealthLogs`.
3. Configure recurring health checks.
4. Expand monitoring to additional Windows servers.
5. Replace the traditional domain service account with a **Group Managed Service Account (gMSA)**.
6. Compare manual service-account password management with automatic gMSA password management.

## Planned Progression

```text
Local Service
     │
     │ COMPLETED
     ▼
Domain Service Account
     │
     │ PARTIALLY COMPLETED
     ▼
Group Managed Service Account (gMSA)
     │
     │ FUTURE LAB
     ▼
Automated Domain-Authenticated Monitoring
```

***

## Project Status

**Status:** Partially completed / troubleshooting documented  
**Next Phase:** Group Managed Service Accounts (gMSA) after resolving the traditional service account logon configuration
