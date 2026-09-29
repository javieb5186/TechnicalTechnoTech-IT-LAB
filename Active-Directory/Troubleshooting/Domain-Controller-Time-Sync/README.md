# Troubleshooting AD DS Domain Controller Time Synchronization

## Project Overview

This troubleshooting lab documents the investigation and resolution of an **Active Directory Domain Services domain controller health issue** involving Windows Time, the PDC Emulator, DC advertising, and Hyper-V time synchronization.

The issue was discovered during a routine domain controller health check using `dcdiag`.

The initial health check reported failures for:

- Advertising
- LocatorCheck
- DFSREvent
- KccEvent
- SystemLog

Further investigation showed that the **Advertising** and **LocatorCheck** failures were related to time synchronization on **DC01**, which holds the **PDC Emulator FSMO role**.

The problem was traced through Windows Time (`W32Time`), NTP configuration, and Hyper-V Integration Services before being successfully resolved.

---

# Lab Environment

| System | Purpose |
|---|---|
| DC01 | Domain Controller / PDC Emulator |
| DC02 | Additional Domain Controller |
| MGMT01 | Administrative Management Workstation |
| CLIENT01 | Domain-Joined Windows Client |
| Hyper-V Host | Hosts the virtual lab environment |

**Domain:** `technicaltechnotech.com`  
**NetBIOS Name:** `TTT`

Both DC01 and DC02 are writable domain controllers.

---

# Troubleshooting Scenario

Before beginning additional domain controller projects, I performed a health check to establish a known-good baseline for the environment.

The primary tools used were:

```text
dcdiag
repadmin
PowerShell
w32tm
Hyper-V PowerShell
```

The goal was to verify:

```text
Domain Controller Health
        +
AD DS Replication
        +
DNS / DC Locator
        +
Windows Time
```

---

# Step 1 — Verify Domain Controllers

From MGMT01, I confirmed the available domain controllers:

```powershell
Get-ADDomainController -Filter * |
    Select-Object Name,IPv4Address,Site,IsGlobalCatalog
```

This confirmed the two-DC environment:

```text
technicaltechnotech.com
          |
     +----+----+
     |         |
   DC01       DC02
     |         |
     +----↔----+
      Replication
```

---

# Step 2 — Check AD DS Replication

Replication health was checked using:

```cmd
repadmin /replsummary
```

For more detailed replication information:

```cmd
repadmin /showrepl
```

These commands help determine whether domain controllers are successfully exchanging Active Directory changes.

### Memory Hook

```text
repadmin = Are my DCs replicating?
```

---

# Step 3 — Run Domain Controller Diagnostics

Because the health check was being performed remotely from MGMT01, each domain controller could be explicitly targeted.

Example:

```cmd
dcdiag /s:DC01
```

A quieter diagnostic was also performed:

```cmd
dcdiag /s:DC01 /q
```

The `/q` parameter only displays errors rather than printing the complete diagnostic output.

The diagnostic reported failures involving:

```text
Advertising
DFSREvent
KccEvent
SystemLog
LocatorCheck
```

Instead of immediately changing configuration based on every warning, the individual failures were investigated.

---

# Step 4 — Isolate the Advertising Failure

The Advertising test was run independently:

```cmd
dcdiag /s:DC01 /test:Advertising
```

DC01 passed connectivity testing but failed Advertising.

The important warning indicated that:

```text
DC01 was not advertising as a time server.
```

This suggested that the Advertising failure was related to Windows Time rather than a general failure of the domain controller.

---

# Step 5 — Investigate LocatorCheck

The LocatorCheck test was then isolated:

```cmd
dcdiag /s:DC01 /test:LocatorCheck
```

The diagnostic reported issues including:

```text
Time server could not be located
PDC role reported unavailable
Good time server could not be located
```

This provided another indication that the underlying problem involved the domain's time hierarchy.

---

# Step 6 — Identify the PDC Emulator

The PDC Emulator was identified using:

```powershell
Get-ADDomain |
    Select-Object PDCEmulator
```

The environment reported:

```text
PDC Emulator = DC01
```

The FSMO roles can also be checked with:

```cmd
netdom query fsmo
```

This was important because the PDC Emulator has a special role in Windows domain time synchronization.

---

# Understanding the AD DS Time Hierarchy

Domain-joined systems normally synchronize time through the Active Directory hierarchy.

A simplified design is:

```text
External NTP Source
        |
        v
DC01 — PDC Emulator
        |
        v
Other Domain Controllers
        |
        v
Domain Members
```

The PDC Emulator at the top of the domain time hierarchy should have a reliable upstream time source.

### Memory Hook

```text
PDC Emulator = AD's main clock
```

---

# Step 7 — Investigate Windows Time

Windows Time status was checked on DC01:

```cmd
w32tm /query /status
```

The important results included:

```text
Leap Indicator: 3 (not synchronized)
Source: VM IC Time Synchronization Provider
```

`Leap Indicator: 3` showed that Windows considered the system clock **unsynchronized**.

The current time source showed that DC01 was receiving time through the Hyper-V integration service.

---

# Step 8 — Inspect Windows Time Configuration

The configuration was checked with:

```cmd
w32tm /query /configuration
```

The relevant configuration initially showed:

```text
Type: NT5DS
NtpServer: local
```

`NT5DS` means that Windows Time attempts to synchronize through the Active Directory domain hierarchy.

This is appropriate for normal domain members and additional domain controllers.

However, DC01 was the **PDC Emulator at the top of the domain hierarchy**.

The situation was effectively:

```text
DC01
PDC Emulator
     |
     v
Attempting domain hierarchy synchronization
     |
     v
No suitable upstream domain source
```

---

# Step 9 — Inspect NTP Peers

Configured peers were examined using:

```cmd
w32tm /query /peers
```

and:

```cmd
w32tm /query /peers /verbose
```

The peer initially showed a state similar to:

```text
State: Pending
```

Information about the previous synchronization attempt indicated that a usable time source had not been successfully established.

---

# Step 10 — Configure an External NTP Source

DC01 was configured to use an external NTP source:

```cmd
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /reliable:yes /update
```

Windows Time was restarted:

```powershell
Restart-Service W32Time
```

A synchronization rediscovery was then requested:

```cmd
w32tm /resync /rediscover
```

The command completed successfully.

The configuration now showed:

```text
NtpServer: time.windows.com,0x8
```

The peer also changed to an active state.

However, DC01 continued reporting:

```text
Source: VM IC Time Synchronization Provider
```

and:

```text
Leap Indicator: 3
```

This showed that configuring an NTP peer alone had not completely resolved the issue.

---

# Step 11 — Test NTP Connectivity

Before making additional changes, communication with the NTP server was tested:

```cmd
w32tm /stripchart /computer:time.windows.com /dataonly /samples:5
```

The test successfully returned time offsets.

An observed offset was approximately:

```text
-0.79 seconds
```

This demonstrated that:

```text
DNS resolution           = Working
Network connectivity     = Working
NTP responses            = Working
Configured peer          = Active
Windows synchronization  = Still incorrect
```

This was an important troubleshooting step because it prevented incorrectly assuming that the external NTP server was unreachable.

---

# Step 12 — Investigate Hyper-V Time Synchronization

Because DC01 continued using:

```text
VM IC Time Synchronization Provider
```

the Hyper-V host configuration was checked.

On the Hyper-V host:

```powershell
Get-VMIntegrationService -VMName "DC01" |
    Where-Object Name -eq "Time Synchronization"
```

The result showed:

```text
Enabled: True
```

Hyper-V Time Synchronization was therefore active for DC01.

The troubleshooting path had now narrowed to:

```text
External NTP reachable
        |
        v
NTP peer active
        |
        v
DC01 still using VMIC
        |
        v
Hyper-V Time Synchronization enabled
```

---

# Step 13 — Disable Hyper-V Time Synchronization for DC01

On the Hyper-V host, the Time Synchronization integration service was disabled for DC01:

```powershell
Disable-VMIntegrationService `
    -VMName "DC01" `
    -Name "Time Synchronization"
```

The change was verified:

```powershell
Get-VMIntegrationService -VMName "DC01" |
    Where-Object Name -eq "Time Synchronization"
```

The service now reported:

```text
Enabled: False
```

This disabled Hyper-V's Time Synchronization integration for DC01 without disabling the Windows Time service inside the virtual machine.

---

# Step 14 — Restart Windows Time and Resynchronize

Inside DC01, Windows Time was restarted:

```powershell
Restart-Service W32Time
```

Time source rediscovery and synchronization were then requested:

```cmd
w32tm /resync /rediscover
```

The Windows Time status was checked again:

```cmd
w32tm /query /source
```

and:

```cmd
w32tm /query /status
```

DC01 was now able to use its configured Windows Time/NTP configuration correctly rather than relying on the Hyper-V VM IC time source.

---

# Step 15 — Verify the Repair

The original failed diagnostics were run again:

```cmd
dcdiag /s:DC01 /test:Advertising
```

and:

```cmd
dcdiag /s:DC01 /test:LocatorCheck
```

Both tests passed.

The troubleshooting process therefore changed the environment from:

```text
Advertising     = FAILED
LocatorCheck    = FAILED
Time Sync       = NOT SYNCHRONIZED
```

to:

```text
Advertising     = PASSED
LocatorCheck    = PASSED
Time Sync       = WORKING
```

---

# Root Cause

The investigation identified a time synchronization problem involving the PDC Emulator.

DC01:

1. Held the PDC Emulator FSMO role.
2. Was not successfully synchronized.
3. Reported `Leap Indicator: 3`.
4. Was using the Hyper-V VM IC Time Synchronization Provider.
5. Was not properly advertising itself as the domain time server.

An external NTP source was configured and verified as reachable.

Hyper-V Time Synchronization was then disabled for DC01 so that Windows Time could use the intended NTP configuration.

After restarting and resynchronizing Windows Time, the original `dcdiag` Advertising and LocatorCheck failures were resolved.

---

# Troubleshooting Workflow

The complete troubleshooting process was:

```text
Routine DC Health Check
        |
        v
    dcdiag /q
        |
        v
Advertising FAILED
LocatorCheck FAILED
        |
        v
Investigate individual tests
        |
        v
DC01 not advertising as time server
        |
        v
Identify PDC Emulator
        |
        v
DC01 = PDC Emulator
        |
        v
Investigate W32Time
        |
        v
Leap Indicator = 3
Source = VM IC Time Synchronization Provider
        |
        v
Inspect NTP configuration
        |
        v
Configure external NTP
        |
        v
NTP Peer = Active
        |
        v
Test NTP connectivity
        |
        v
External NTP reachable
        |
        v
Check Hyper-V integration
        |
        v
Hyper-V Time Synchronization = Enabled
        |
        v
Disable VM Time Synchronization for DC01
        |
        v
Restart W32Time
        |
        v
Resynchronize
        |
        v
Re-run dcdiag
        |
        v
Advertising = PASS
LocatorCheck = PASS
```

---

# Commands Used

## Active Directory

```powershell
Get-ADDomainController -Filter *
```

```powershell
Get-ADDomain |
    Select-Object PDCEmulator
```

```cmd
netdom query fsmo
```

---

## Domain Controller Diagnostics

```cmd
dcdiag /s:DC01
```

```cmd
dcdiag /s:DC01 /q
```

```cmd
dcdiag /s:DC01 /test:Advertising
```

```cmd
dcdiag /s:DC01 /test:LocatorCheck
```

---

## Replication

```cmd
repadmin /replsummary
```

```cmd
repadmin /showrepl
```

---

## Windows Time

```cmd
w32tm /query /status
```

```cmd
w32tm /query /source
```

```cmd
w32tm /query /configuration
```

```cmd
w32tm /query /peers
```

```cmd
w32tm /query /peers /verbose
```

```cmd
w32tm /resync /rediscover
```

```cmd
w32tm /stripchart /computer:time.windows.com /dataonly /samples:5
```

---

## Configure NTP

```cmd
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /reliable:yes /update
```

```powershell
Restart-Service W32Time
```

---

## Hyper-V

```powershell
Get-VMIntegrationService -VMName "DC01" |
    Where-Object Name -eq "Time Synchronization"
```

```powershell
Disable-VMIntegrationService `
    -VMName "DC01" `
    -Name "Time Synchronization"
```

---

# Key Troubleshooting Lesson

The original error was:

```text
Advertising FAILED
```

However, that did **not** necessarily mean that AD DS Advertising itself was the root problem.

The investigation revealed:

```text
Advertising failure
        |
        v
DC not advertising proper time service
        |
        v
Windows Time synchronization problem
        |
        v
PDC Emulator using incorrect/unwanted time source
        |
        v
Hyper-V Time Synchronization interaction
```

This demonstrated an important troubleshooting principle:

> Do not stop at the first error message. Trace the failed service through its dependencies until the underlying problem is identified.

---

# What I Learned

- How to perform basic domain controller health checks.
- How to use `dcdiag` to diagnose individual domain controllers.
- How to use `/q` to display only diagnostic failures.
- How to isolate individual `dcdiag` tests.
- How to check Active Directory replication with `repadmin`.
- Why the PDC Emulator is important to domain time synchronization.
- How Windows domain members obtain time through the AD hierarchy.
- How to inspect Windows Time with `w32tm`.
- What `Leap Indicator: 3` means.
- How to inspect configured NTP peers.
- The difference between `NT5DS` domain hierarchy synchronization and manually configured NTP.
- How to test NTP communication independently of Windows Time synchronization.
- How Hyper-V Integration Services can provide time to virtual machines.
- How to inspect and disable Hyper-V Time Synchronization for a VM.
- How to verify a repair by rerunning the original failed diagnostic tests.
- Why successful troubleshooting requires identifying the root cause rather than simply responding to the first error.

---

# Skills Practiced

- Active Directory Domain Services
- Domain Controller Administration
- AD DS Troubleshooting
- Domain Controller Health Checks
- `dcdiag`
- `repadmin`
- Active Directory Replication
- FSMO Roles
- PDC Emulator
- Windows Time Service
- `w32tm`
- NTP
- DC Locator
- PowerShell
- Hyper-V
- Hyper-V Integration Services
- Network Service Troubleshooting
- Root Cause Analysis
- Verification and Validation

---

# Suggested Screenshots

## Screenshot 1 — Initial `dcdiag` Failure

Capture:

```cmd
dcdiag /s:DC01 /q
```

showing the Advertising and LocatorCheck failures.

---

## Screenshot 2 — PDC Emulator

Capture:

```powershell
Get-ADDomain |
    Select-Object PDCEmulator
```

showing:

```text
DC01
```

---

## Screenshot 3 — Initial Windows Time Status

Capture:

```cmd
w32tm /query /status
```

showing:

```text
Leap Indicator: 3
Source: VM IC Time Synchronization Provider
```

This is one of the most useful screenshots because it shows the condition being investigated.

---

## Screenshot 4 — NTP Peer

Capture:

```cmd
w32tm /query /peers /verbose
```

showing:

```text
time.windows.com
State: Active
```

---

## Screenshot 5 — NTP Connectivity Test

Capture:

```cmd
w32tm /stripchart /computer:time.windows.com /dataonly /samples:5
```

showing successful time responses.

This demonstrates that NTP connectivity was tested independently before changing the Hyper-V configuration.

---

## Screenshot 6 — Hyper-V Time Synchronization

Capture:

```powershell
Get-VMIntegrationService -VMName "DC01" |
    Where-Object Name -eq "Time Synchronization"
```

showing the Time Synchronization integration service configuration.

---

## Screenshot 7 — Successful Verification

Capture the final:

```cmd
dcdiag /s:DC01 /test:Advertising
```

and:

```cmd
dcdiag /s:DC01 /test:LocatorCheck
```

showing that both tests passed.

This provides clear **before-and-after evidence** that the issue was resolved.

---

# Project Summary

This project began as a routine Active Directory domain controller health check and developed into a troubleshooting exercise involving the PDC Emulator, Windows Time, NTP, DC Locator, and Hyper-V.

Rather than treating the initial `dcdiag` failures as isolated problems, each failure was investigated until a common dependency was identified.

The final troubleshooting process demonstrated:

```text
Detect
  ↓
Isolate
  ↓
Investigate
  ↓
Test
  ↓
Identify Root Cause
  ↓
Remediate
  ↓
Verify
```

The result was a healthy DC01 that successfully passed the previously failing **Advertising** and **LocatorCheck** diagnostic tests.
