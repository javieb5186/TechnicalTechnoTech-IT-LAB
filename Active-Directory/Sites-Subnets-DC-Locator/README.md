# AD DS Sites, Subnets, and Domain Controller Locator Lab

## Project Overview

This project explored how **Active Directory Domain Services (AD DS)** uses **Sites, Subnets, DNS, and DC Locator** to help domain-joined computers locate appropriate domain controllers.

The lab started with two domain controllers and multiple IP subnets operating as a single Active Directory site.

I then:

- Identified which domain controller authenticated CLIENT01.
- Used `nltest` to inspect domain controller discovery.
- Examined Active Directory DNS SRV records.
- Created a custom Active Directory site.
- Created subnet objects representing the lab's actual IP networks.
- Verified that CLIENT01 was correctly mapped to its site.
- Created a simulated branch-office site.
- Moved CLIENT01's subnet to the branch site.
- Observed how a client behaves when its site does not contain a domain controller.
- Explored what would be required to place DC02 into the branch site correctly.
- Restored the environment to its original network/site design.

The project demonstrated that Active Directory does not normally require a traditional load balancer between domain controllers.

Instead, clients use **DNS, DC Locator, Sites, and Subnets** to discover appropriate domain controllers.

---

# Lab Environment

| System | IP Address | Subnet | Purpose |
|---|---|---|---|
| DC01 | `10.20.10.10` | `10.20.10.0/24` | Domain Controller |
| DC02 | `10.20.10.11` | `10.20.10.0/24` | Additional Domain Controller |
| CLIENT01 | `10.20.20.10` | `10.20.20.0/24` | Domain-Joined Client |
| MGMT01 | `10.20.30.10` | `10.20.30.0/24` | Management Workstation |

**Subnet Mask:** `255.255.255.0`

The default gateway for each subnet uses the first usable address:

```text
10.20.10.1
10.20.20.1
10.20.30.1
```

**Domain:** `technicaltechnotech.com`

**NetBIOS Domain:** `TTT`

---

# Initial Network Design

The lab uses three separate `/24` IP networks:

```text
DC Network
10.20.10.0/24
│
├── DC01
│   10.20.10.10
│
└── DC02
    10.20.10.11


Client Network
10.20.20.0/24
│
└── CLIENT01
    10.20.20.10


Management Network
10.20.30.0/24
│
└── MGMT01
    10.20.30.10
```

Although these are separate IP subnets, they are part of the same lab location.

---

# Part 1 — Determine Which DC CLIENT01 Uses

I logged into CLIENT01 with a domain account and checked the domain controller associated with the logon session.

```cmd
echo %LOGONSERVER%
```

CLIENT01 reported:

```text
\\DC02
```

This showed that the current user's logon session had authenticated using DC02.

### Important

`%LOGONSERVER%` shows the domain controller associated with the user's current logon session.

It does not mean that the computer will permanently use that domain controller for every Active Directory operation.

---

# Part 2 — Use DC Locator

I then used `nltest` to ask Windows to locate a domain controller.

```cmd
nltest /dsgetdc:technicaltechnotech.com
```

The command returned information including:

```text
DC
Address
Domain Name
Forest Name
DC Site Name
Our Site Name
Flags
```

I also listed available domain controllers:

```cmd
nltest /dclist:technicaltechnotech.com
```

This demonstrated that Windows can discover the domain controllers available for the domain.

---

# Logon Server vs DC Locator

An important distinction from this lab was:

```text
%LOGONSERVER%
      ↓
Which DC handled this user's logon?


nltest /dsgetdc
      ↓
Which DC does DC Locator return for this request?
```

These values do not necessarily have to be the same.

### Memory Hook

```text
LOGONSERVER = who logged me in

DSGETDC = find me a DC
```

---

# Part 3 — Examine Active Directory DNS Records

Active Directory relies heavily on DNS to advertise domain services.

On CLIENT01, I opened `nslookup`:

```cmd
nslookup
```

I changed the query type to SRV:

```text
set type=SRV
```

Then queried:

```text
_ldap._tcp.dc._msdcs.technicaltechnotech.com
```

These DNS SRV records advertise domain controllers capable of providing LDAP/domain services.

Conceptually:

```text
CLIENT01
    |
    | Needs a domain controller
    v
DNS
    |
    | Queries AD SRV records
    v
_ldap._tcp.dc._msdcs.technicaltechnotech.com
    |
    +----------------+
    |                |
    v                v
  DC01             DC02
```

---

# Understanding SRV Records

Normal DNS records commonly map names to IP addresses.

For example:

```text
A Record

DC01
  ↓
10.20.10.10
```

SRV records provide information about **services**.

For example:

```text
LDAP service
     ↓
Which servers provide it?
     ↓
DC01 / DC02
```

This is one of the mechanisms Active Directory uses for domain controller discovery.

---

# Part 4 — Active Directory Sites

I opened:

```text
Active Directory Sites and Services
```

The domain controllers initially existed under:

```text
Default-First-Site-Name
```

I created a new site:

```text
HQ-Site
```

using:

```text
DEFAULTIPSITELINK
```

I then moved:

```text
DC01
DC02
```

into:

```text
HQ-Site
```

The resulting structure was:

```text
Sites
│
└── HQ-Site
    │
    └── Servers
        ├── DC01
        └── DC02
```

---

# What an Active Directory Site Represents

An Active Directory Site represents a **well-connected network location**.

A site might represent:

```text
Headquarters
Branch Office
Datacenter
Regional Office
```

A site is not the same thing as:

```text
Domain
OU
VLAN
Subnet
```

Instead, sites allow AD DS to understand the physical/network topology of the environment.

---

# Part 5 — Create Active Directory Subnet Objects

The lab already had real IP networks:

```text
10.20.10.0/24
10.20.20.0/24
10.20.30.0/24
```

I created corresponding subnet objects in **Active Directory Sites and Services**.

Initially, all three were assigned to:

```text
HQ-Site
```

Result:

```text
10.20.10.0/24 → HQ-Site
10.20.20.0/24 → HQ-Site
10.20.30.0/24 → HQ-Site
```

The topology could therefore be represented as:

```text
                    HQ-Site
                       |
          +------------+------------+
          |            |            |
          v            v            v
    10.20.10.0/24 10.20.20.0/24 10.20.30.0/24
          |            |            |
      DC01/DC02     CLIENT01      MGMT01
```

---

# Network Subnet vs AD Subnet

One of the most important concepts from this lab was the difference between an actual network subnet and an AD DS subnet object.

## Network Subnet

Example:

```text
10.20.20.0/24
```

This is part of the actual IP network design.

It determines which addresses belong to the network.

## Active Directory Subnet Object

The AD subnet object tells Active Directory:

```text
10.20.20.0/24
       ↓
belongs to
       ↓
HQ-Site
```

Creating an AD subnet does **not** create the actual IP network.

It simply maps an existing network to an Active Directory site.

### Memory Hook

> **Network subnet = actual network. AD subnet = map the network to an AD site.**

---

# Part 6 — Verify CLIENT01's Site

On CLIENT01, I ran:

```cmd
nltest /dsgetsite
```

CLIENT01 returned:

```text
HQ-Site
```

This demonstrated how Active Directory determined the client's site.

Conceptually:

```text
CLIENT01
10.20.20.10
     |
     v
What subnet contains this address?
     |
     v
10.20.20.0/24
     |
     v
Which AD site owns that subnet?
     |
     v
HQ-Site
```

### Memory Hook

> **Subnet tells AD where the client is; Site tells AD which DCs are nearby.**

---

# Part 7 — Simulate a Branch Office

To better understand site-aware domain controller discovery, I created another site:

```text
Branch-Site
```

using:

```text
DEFAULTIPSITELINK
```

The topology initially became:

```text
Sites
│
├── HQ-Site
│   ├── DC01
│   └── DC02
│
└── Branch-Site
```

Branch-Site initially contained no domain controllers.

---

# Part 8 — Move CLIENT01's Subnet to Branch-Site

I changed the Active Directory subnet mapping for:

```text
10.20.20.0/24
```

from:

```text
HQ-Site
```

to:

```text
Branch-Site
```

The new topology was:

```text
HQ-Site
│
├── 10.20.10.0/24
│   ├── DC01
│   └── DC02
│
└── 10.20.30.0/24
    └── MGMT01


Branch-Site
│
└── 10.20.20.0/24
    └── CLIENT01
```

This simulated a branch office that did not have its own domain controller.

---

# Part 9 — Refresh DC Locator Information

Initially, CLIENT01 continued reporting:

```text
HQ-Site
```

even though its subnet had been reassigned.

This demonstrated that site/DC Locator information can be cached.

I forced domain controller discovery:

```cmd
nltest /dsgetdc:technicaltechnotech.com /force
```

When necessary, I restarted the Netlogon service from an elevated PowerShell session:

```powershell
Restart-Service Netlogon
```

Then ran:

```cmd
nltest /dsgetdc:technicaltechnotech.com /force
```

followed by:

```cmd
nltest /dsgetsite
```

CLIENT01 then correctly reported:

```text
Branch-Site
```

---

# Verify AD Subnet Configuration

The configured Active Directory subnet mappings can also be inspected using PowerShell:

```powershell
Get-ADReplicationSubnet -Filter * |
    Select-Object Name,Site
```

During the branch simulation, the intended mapping was:

```text
10.20.10.0/24 → HQ-Site
10.20.20.0/24 → Branch-Site
10.20.30.0/24 → HQ-Site
```

---

# Part 10 — Branch Site Without a Domain Controller

At this point, CLIENT01 belonged to:

```text
Branch-Site
```

but Branch-Site did not contain a domain controller.

The design was:

```text
HQ-Site                     Branch-Site

DC01                        CLIENT01
DC02                        10.20.20.10
  |                              |
  |                              |
  +---------- Network -----------+
```

CLIENT01 still needed Active Directory services.

When DC Locator was forced:

```cmd
nltest /dsgetdc:technicaltechnotech.com /force
```

DC Locator returned:

```text
DC01
```

This demonstrated an important concept:

> **An Active Directory site does not have to contain a domain controller.**

If there is no suitable site-local DC, clients can locate a domain controller elsewhere in the Active Directory topology.

---

# Part 11 — Explore Site-Local Domain Controller Design

The next experiment was to place DC02 into:

```text
Branch-Site
```

Conceptually, the goal was:

```text
HQ-Site                     Branch-Site
   |                             |
 DC01                          DC02
                                 |
                              CLIENT01
```

This would allow CLIENT01 to have a site-local domain controller.

However, the experiment exposed an important configuration issue.

DC02's actual IP address was:

```text
10.20.10.11
```

which belongs to:

```text
10.20.10.0/24
```

and that subnet belongs to:

```text
HQ-Site
```

Therefore, simply moving the DC02 server object into Branch-Site created an inconsistent topology:

```text
DC02 Server Object
        ↓
Branch-Site

BUT

DC02 IP
10.20.10.11
        ↓
10.20.10.0/24
        ↓
HQ-Site
```

This was not a good representation of a properly designed production environment.

---

# Correct Branch Office Design

A cleaner design would place DC02 on the actual Branch-Site network.

For example:

```text
HQ-Site
10.20.10.0/24
     |
     └── DC01
         10.20.10.10


Branch-Site
10.20.20.0/24
     |
     ├── DC02
     |   10.20.20.11
     |
     └── CLIENT01
         10.20.20.10
```

Now everything aligns:

```text
CLIENT01 IP
     ↓
10.20.20.0/24
     ↓
Branch-Site


DC02 IP
     ↓
10.20.20.0/24
     ↓
Branch-Site
```

DC Locator could then use the site's topology to help CLIENT01 discover the site-local domain controller.

---

# Why We Did Not Reconfigure DC02

Reconfiguring a domain controller's network configuration was unnecessary for demonstrating the underlying AD DS concept.

Instead, the simulation was stopped after identifying the topology mismatch.

This avoided making unnecessary infrastructure changes solely for the experiment.

---

# Part 12 — Restore the Lab

DC02 was moved back to:

```text
HQ-Site
```

The CLIENT01 subnet:

```text
10.20.20.0/24
```

was mapped back to:

```text
HQ-Site
```

The final lab topology returned to:

```text
                         HQ-Site
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
    10.20.10.0/24     10.20.20.0/24     10.20.30.0/24
          |                 |                 |
     +----+----+         CLIENT01           MGMT01
     |         |
   DC01       DC02
```

This accurately represents the current lab as one well-connected location containing multiple IP subnets.

---

# How DC Locator Works — Simplified

The overall process can be summarized as:

```text
CLIENT01 needs Active Directory
             |
             v
       DNS / SRV Records
             |
             v
         DC Locator
             |
             v
What AD site is CLIENT01 in?
             |
             v
CLIENT01 IP = 10.20.20.10
             |
             v
Subnet = 10.20.20.0/24
             |
             v
Site = HQ-Site
             |
             v
Find an appropriate DC
             |
        +----+----+
        |         |
        v         v
      DC01       DC02
```

---

# Domain Controllers and Load Balancing

This lab also clarified an important misconception.

Multiple domain controllers do not normally require a traditional network load balancer such as:

```text
CLIENT
   |
   v
Load Balancer
   |
+--+--+
|     |
DC01 DC02
```

Instead, Active Directory uses mechanisms including:

```text
DNS
 +
SRV Records
 +
DC Locator
 +
Sites
 +
Subnets
```

to help clients locate appropriate domain controllers.

A simplified view is:

```text
CLIENT01
    |
    v
DNS / DC Locator
    |
    v
AD Site Information
    |
    +--------+
    |        |
    v        v
  DC01      DC02
```

---

# Commands Used

## Determine Logon Server

```cmd
echo %LOGONSERVER%
```

---

## Locate a Domain Controller

```cmd
nltest /dsgetdc:technicaltechnotech.com
```

Force rediscovery:

```cmd
nltest /dsgetdc:technicaltechnotech.com /force
```

---

## List Domain Controllers

```cmd
nltest /dclist:technicaltechnotech.com
```

---

## Determine Client Site

```cmd
nltest /dsgetsite
```

---

## Restart Netlogon

```powershell
Restart-Service Netlogon
```

---

## View Active Directory Subnets

```powershell
Get-ADReplicationSubnet -Filter * |
    Select-Object Name,Site
```

---

## Query Active Directory DNS SRV Records

```cmd
nslookup
```

Then:

```text
set type=SRV
_ldap._tcp.dc._msdcs.technicaltechnotech.com
```

---

# Troubleshooting Workflow

During the Branch-Site experiment, CLIENT01 initially continued reporting HQ-Site after its subnet had been reassigned.

The troubleshooting process was:

```text
Change AD subnet mapping
        |
        v
10.20.20.0/24 → Branch-Site
        |
        v
nltest /dsgetsite
        |
        v
Still reports HQ-Site
        |
        v
Force DC Locator rediscovery
        |
        v
Refresh Netlogon when necessary
        |
        v
nltest /dsgetsite
        |
        v
Branch-Site
```

This demonstrated that changing Active Directory configuration does not necessarily mean every client immediately reflects the change.

Cached information may need to be refreshed.

---

# Key Design Lesson

The most important lesson from the branch-office simulation was:

> **Active Directory Sites should reflect the actual network topology.**

Simply moving a domain controller object into another AD site does not physically move that server onto the site's network.

A good design aligns:

```text
Physical / Logical Network
          +
IP Subnets
          +
AD Subnet Objects
          +
AD Sites
          +
Domain Controller Placement
```

For example:

```text
Branch Network
10.20.20.0/24
      |
      v
AD Subnet Object
10.20.20.0/24
      |
      v
Branch-Site
      |
      v
Branch DC
10.20.20.11
```

---

# What I Learned

- How domain-joined computers discover domain controllers.
- How to determine which DC handled a user's logon.
- How to use `nltest` to investigate DC Locator.
- The difference between `%LOGONSERVER%` and `nltest /dsgetdc`.
- Why Active Directory depends heavily on DNS.
- How DNS SRV records advertise Active Directory services.
- What an Active Directory Site represents.
- How Active Directory subnet objects work.
- The difference between an IP subnet and an AD subnet object.
- How multiple IP subnets can belong to one Active Directory site.
- How a client's IP address determines its Active Directory site.
- How to verify a computer's AD site using `nltest /dsgetsite`.
- How cached DC Locator/site information can affect troubleshooting.
- How restarting Netlogon can refresh discovery information.
- How clients can locate domain controllers outside their site when no site-local DC exists.
- Why domain controller placement should correspond with the actual network topology.
- Why moving a DC's AD Sites and Services object does not change its actual network location.
- How Sites and Subnets help Active Directory provide location-aware domain controller discovery.
- Why traditional network load balancing is generally not how AD DS distributes domain controller usage.

---

# Skills Practiced

- Active Directory Domain Services
- Active Directory Sites and Services
- AD Sites
- AD Subnets
- Domain Controller Administration
- DC Locator
- DNS
- DNS SRV Records
- Netlogon
- `nltest`
- `nslookup`
- PowerShell
- Active Directory PowerShell
- IP Addressing
- Subnetting
- Multi-Subnet Network Design
- Domain Controller Discovery
- Domain Controller Placement
- AD DS Network Topology
- Troubleshooting
- Infrastructure Design

---

# Suggested Screenshots

## Screenshot 1 — Initial DC Discovery

Capture:

```cmd
echo %LOGONSERVER%
```

showing:

```text
\\DC02
```

---

## Screenshot 2 — DC Locator

Capture:

```cmd
nltest /dsgetdc:technicaltechnotech.com
```

showing the returned DC and site information.

---

## Screenshot 3 — DNS SRV Records

Capture the `nslookup` results for:

```text
_ldap._tcp.dc._msdcs.technicaltechnotech.com
```

showing the domain controllers advertised through DNS.

---

## Screenshot 4 — HQ-Site

Capture **Active Directory Sites and Services** showing:

```text
HQ-Site
└── Servers
    ├── DC01
    └── DC02
```

---

## Screenshot 5 — AD Subnets

Capture the Subnets section showing:

```text
10.20.10.0/24
10.20.20.0/24
10.20.30.0/24
```

---

## Screenshot 6 — CLIENT01 Site

Capture:

```cmd
nltest /dsgetsite
```

showing:

```text
HQ-Site
```

---

## Screenshot 7 — Branch-Site Simulation

Capture Active Directory Sites and Services while:

```text
10.20.20.0/24 → Branch-Site
```

This demonstrates the temporary branch-office topology.

---

## Screenshot 8 — Branch Client Site

Capture:

```cmd
nltest /dsgetsite
```

showing:

```text
Branch-Site
```

---

# Project Summary

This project demonstrated how Active Directory Domain Services understands the physical/network topology of an organization.

The main relationship is:

```text
IP Address
    ↓
Network Subnet
    ↓
AD Subnet Object
    ↓
AD Site
    ↓
DC Locator
    ↓
Appropriate Domain Controller
```

Rather than relying on a traditional load balancer, Active Directory uses **DNS, SRV records, DC Locator, Sites, and Subnets** to help clients locate domain controllers.

The branch-office simulation also demonstrated that AD Sites and Subnets should represent the **real network design** rather than being treated as arbitrary administrative containers.

### Final Memory Hook

> **Subnet tells AD where the client is. Site tells AD which DCs are nearby.**
