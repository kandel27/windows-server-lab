# Reference Sheet

Names, addresses, specifications and dates.

> **No credentials in this document.** Every password lives in a password manager only. This
> sheet records what things are called and where they are, never how to sign in to them.

---

## Identity

| Item | Value |
|---|---|
| Domain (FQDN) | `kandel-lab.internal` |
| NetBIOS name | `KANDEL-LAB` |
| Domain fixed | Before promotion, deliberately |
| Forest / domain functional level | Windows Server 2025 |

## Network

| Item | Value |
|---|---|
| Lab subnet | `192.168.50.0/24` |
| VirtualBox network | `LabNet`, internal network |
| `DC01` static address | `192.168.50.10` |
| Subnet mask | `255.255.255.0` |
| DNS server | `192.168.50.10` — `DC01`, pointing at itself |
| Default gateway | None. `LabNet` is isolated |
| DHCP scope | `192.168.50.100` – `192.168.50.200`, named `LabNet` |
| DHCP server | `DC01`, authorised in Active Directory, bound to `LabNet` only |
| DHCP options | DNS `192.168.50.10`, parent domain `kandel-lab.internal`, no router option |
| Internet access | `DC01` adapter 2, NAT, for updates and activation only |

## Machines

| Name | Type | Role | Address |
|---|---|---|---|
| Hypervisor host | Physical laptop | VirtualBox host | Home Wi-Fi, DHCP |
| Managed endpoint | Physical desktop | Entra ID joined, Intune enrolled | Home Wi-Fi, DHCP |
| `DC01` | VM — Windows Server 2025 | AD DS, DNS, DHCP | `192.168.50.10`, static |
| `WIN11-01` | VM — Windows 11 Enterprise | Domain-joined client | `192.168.50.100`, DHCP from `DC01` |

## `DC01` virtual machine specification

| Setting | Value |
|---|---|
| Hypervisor | VirtualBox 7.2.18 |
| Edition | Windows Server 2025 Standard Evaluation (Desktop Experience) |
| Memory | 4096 MB |
| Processors | 2 |
| Disk | 60 GB, VDI, dynamically allocated |
| Adapter 1 | Internal network — `LabNet` |
| Adapter 2 | NAT |
| Boot order | Optical, Hard Disk, Floppy |

## Evaluation expiry

Windows Server 2025 evaluation runs for 180 days from installation, as shown in the desktop
watermark. Recording the dates here stops a lab expiring part-way through a project.

| Item | Evaluation period | Expires |
|---|---|---|
| `DC01` — Windows Server 2025 Evaluation | 180 days | March 2027 |
| `WIN11-01` — Windows 11 Enterprise Evaluation | 90 days | December 2026 |

## Directory structure

| Item | Value |
|---|---|
| Top-level OU | `KANDEL-LAB` |
| Child OUs | `Users`, `Computers`, `Groups` |
| Computer objects | `WIN11-01` moved from the default Computers container into `KANDEL-LAB\Computers` |
| User accounts | Created in `KANDEL-LAB\Users` rather than the default Users container, so Group Policy can be linked |
| Group Policy | Default Domain Controllers Policy left at its defaults. Ordinary users sign in at `WIN11-01`, not at `DC01` |

---

Update this sheet whenever an address, name or date changes. A reference sheet that is out of
date is worse than none, because it is trusted.
