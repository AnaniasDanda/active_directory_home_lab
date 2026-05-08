# Active Directory Home Lab

A fully functional on-premises Active Directory environment built from scratch using virtualization. This lab simulates a real-world enterprise network with a Windows Server Domain Controller, domain-joined clients, DHCP, DNS, and Group Policy enforcement.

---

## Lab Overview

| Component | Details |
|---|---|
| **Domain Controller** | Windows Server 2019/2022 — AD DS, DNS, DHCP |
| **Client Machines** | Windows 10/11 — domain-joined |
| **Domain** | `lab.local` |
| **Subnet** | `192.168.10.0/24` |

---

## Skills Demonstrated

- Deploying and promoting a Windows Server Domain Controller
- Configuring AD DS, DNS, and DHCP server roles
- Joining Windows clients to an Active Directory domain
- Creating and managing Organizational Units (OUs), users, and security groups
- Authoring and enforcing Group Policy Objects (GPOs)
- Static IP and network configuration in a virtual environment

---

## Environment Requirements

**Software**
- VirtualBox or VMware Workstation Player
- Windows Server 2019/2022 ISO (Evaluation edition is sufficient)
- Windows 10/11 ISO

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        lab.local Domain                     │
│                                                             │
│   ┌─────────────────────┐        ┌─────────────────────┐   │
│   │        DC01         │        │        PC01         │   │
│   │  Windows Server     │◄──────►│   Windows 10/11     │   │
│   │  192.168.10.10      │        │  192.168.10.100–200 │   │
│   │                     │        │    (DHCP lease)     │   │
│   │  Roles:             │        └─────────────────────┘   │
│   │  • AD DS            │                                   │
│   │  • DNS              │        ┌─────────────────────┐   │
│   │  • DHCP             │◄──────►│        PC02         │   │
│   └─────────────────────┘        │   Windows 10/11     │   │
│                                  │  192.168.10.100–200 │   │
│                                  └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Build Walkthrough

### Part A — Domain Controller Setup

#### Step 1 — Create the DC01 Virtual Machine

| Setting | Value |
|---|---|
| VM Name | `DC01` |
| CPU | 2 cores |
| RAM | 4–8 GB |
| Disk | 60 GB |
| Network Adapter | Host-Only or Internal Network (add NAT adapter for internet access) |

#### Step 2 — Install Windows Server

1. Boot from the Windows Server ISO
2. Select **Windows Server Standard (Desktop Experience)**
3. Set a strong local administrator password
4. Complete installation and log in

#### Step 3 — Configure a Static IP

Navigate to: `Control Panel → Network and Internet → Network Connections`

Right-click the Ethernet adapter → **Properties** → **IPv4**

| Field | Value |
|---|---|
| IP Address | `192.168.10.10` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.10.1` |
| Preferred DNS | `192.168.10.10` *(DC points to itself)* |

#### Step 4 — Rename the Server

`Server Manager → Local Server → Computer Name → Change`

Rename to `DC01`, then reboot.

#### Step 5 — Install AD DS and DNS Roles

`Server Manager → Add Roles and Features → Role-based Installation`

Select the following roles:
- **Active Directory Domain Services**
- **DNS Server**

Proceed through the wizard and install.

#### Step 6 — Promote to Domain Controller

1. In Server Manager, click the **notification flag** → *Promote this server to a domain controller*
2. Choose **Add a new forest**
3. Set the root domain name: `lab.local`
4. Configure a DSRM (Directory Services Restore Mode) password and store it securely
5. Leave DNS delegation defaults unchanged
6. Complete the wizard — the server will reboot automatically

#### Step 7 — Verify AD DS and DNS

After the reboot, confirm the following:

- **Active Directory Users and Computers (ADUC)** — the `lab.local` domain is visible
- **DNS Manager** — forward lookup zone `lab.local` is present, along with `_msdcs` zone entries

---

### Part B — Client VM Setup and Domain Join

#### Step 8 — Create the Client VM (PC01)

| Setting | Value |
|---|---|
| VM Name | `PC01` |
| CPU | 2 cores |
| RAM | 4 GB |
| Disk | 60 GB |
| Network | Same internal network as DC01 |

Install Windows 10 or 11.

#### Step 9 — Configure Client Network Settings

Set a static IP (or use DHCP once configured):

| Field | Value |
|---|---|
| IP Address | `192.168.10.20` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.10.1` |
| Preferred DNS | `192.168.10.10` *(must point to DC)* |

> **Critical:** The DNS field must point to the Domain Controller. This is the most common point of failure when joining a domain.

#### Step 10 — Verify Connectivity

From PC01 Command Prompt:

```cmd
ping 192.168.10.10
nslookup lab.local
```

Both should succeed before proceeding.

#### Step 11 — Join the Domain

`Settings → System → About → Domain or workgroup → Join domain`

- Domain: `lab.local`
- Credentials: `LAB\Administrator`
- Reboot, then log in as a domain user

---

### Part C — DHCP Configuration

#### Step 12 — Install the DHCP Server Role on DC01

`Server Manager → Add Roles and Features → DHCP Server`

After installation, complete the post-install DHCP configuration wizard.

#### Step 13 — Create a DHCP Scope

Open the **DHCP console** → `IPv4 → New Scope`

| Option | Value |
|---|---|
| Address Range | `192.168.10.100 – 192.168.10.200` |
| Subnet Mask | `255.255.255.0` |
| Router (Option 003) | `192.168.10.1` |
| DNS Server (Option 006) | `192.168.10.10` |
| Domain Name (Option 015) | `lab.local` |

Activate the scope.

#### Step 14 — Switch Clients to DHCP

On PC01, set the NIC to obtain an IP automatically, then renew:

```cmd
ipconfig /release
ipconfig /renew
```

---

### Part D — Active Directory Administration

#### Step 15 — Create Organizational Units

In **Active Directory Users and Computers**, right-click the domain → `New → Organizational Unit`

Create the following OUs:
- `OU=Users`
- `OU=Computers`
- `OU=Admins` *(optional)*

#### Step 16 — Create Users and Security Groups

**Users** (in the Users OU):
- `jdoe` — standard domain user

**Groups:**
- `GG-IT-Admins`
- `GG-Staff`

Add users to appropriate groups.

#### Step 17 — Create and Test a Group Policy Object

1. Open **Group Policy Management**
2. Right-click the `Users` OU → *Create a GPO in this domain and link it here*
3. Name it (e.g., `Block-ControlPanel`)
4. Edit the GPO:

```
User Configuration
  └── Administrative Templates
        └── Control Panel
              └── Prohibit access to Control Panel and PC settings → Enabled
```

**Verify on the client:**

```cmd
gpupdate /force
```

Log in as `jdoe` and confirm that access to Control Panel is blocked.

---

## Key Concepts Covered

| Concept | Implementation |
|---|---|
| Directory Services | Active Directory Domain Services (AD DS) |
| Name Resolution | DNS integrated with AD, forward lookup zone `lab.local` |
| IP Management | DHCP server with scope, options, and lease management |
| Identity Management | OUs, users, groups in a structured hierarchy |
| Policy Enforcement | GPO linked to OUs, verified with `gpupdate /force` |

---

## Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| `nslookup lab.local` fails | DNS not pointing to DC | Set DNS to `192.168.10.10` |
| Domain join fails | DNS or network misconfiguration | Verify `ping` and `nslookup` first |
| GPO not applying | Policy not linked or not refreshed | Run `gpupdate /force`; check GPMC linking |
| Client gets wrong IP | DHCP scope not activated | Activate scope in DHCP console |
| DC01 unreachable from client | Wrong virtual network segment | Ensure both VMs are on the same internal network |

---

## Lab Environment

- **Hypervisor:** VirtualBox / VMware Workstation Player
- **Server OS:** Windows Server 2019 / 2022 (Evaluation)
- **Client OS:** Windows 10 / 11

---

*This lab was built as a hands-on exercise in Windows Server administration, Active Directory architecture, and enterprise network fundamentals.*
