# Build Log

One entry per working session: what was done, what went wrong, how it was resolved, and what
comes next. The problems matter more than the successes, because they are the part that cannot
be copied from a tutorial.

---

## Session 1 — planning and the `DC01` build

**Goal:** finalise the lab design and build `DC01`, the first domain controller.

### What was done

- Audited the available machines for CPU, memory, free disk and virtualisation support, and
  assigned each a fixed role.
- Updated VirtualBox to 7.2.18 and removed old virtual machines.
- Produced the network design as a draw.io diagram.
- Fixed the domain name as `kandel-lab.internal` before any promotion took place.
- Decided the storage split: disk images and ISOs on local storage, documentation in cloud sync.
- Downloaded the Windows Server 2025 and Windows 11 Enterprise evaluation ISOs from the
  Microsoft Evaluation Center.
- Created the `DC01` virtual machine: 4 GB memory, 2 vCPU, 60 GB dynamically allocated disk,
  adapter 1 on the internal network `LabNet`, adapter 2 on NAT.
- Installed Windows Server 2025 Standard Evaluation (Desktop Experience).

---

### Problem 1 — Setup refused to install to the disk

**Symptom**

> There is an error selecting this partition for install. Please select a different partition or
> refresh selections.

**Cause**

The virtual disk already carried a partition left over from earlier use, and Setup would not
install into it.

**Fix**

Deleted the existing partition so the disk presented a single block of unallocated space, then
selected that. Setup then created its own layout: EFI, MSR, Windows and recovery partitions.

**What it taught**

*Delete Partition* is harmless on an empty virtual disk and irreversible on a real one. Confirm
which disk is selected before using it.

---

### Problem 2 — Server Core installed instead of Desktop Experience

**Symptom**

After installation, signing in landed in SConfig over a black screen. No desktop, no taskbar,
no Server Manager.

**Diagnosis**

```powershell
Test-Path $env:windir\explorer.exe
# False
```

Server Core ships without the Explorer shell, so a `False` here confirms the edition. The
authoritative check is the `InstallationType` value under:

```
HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion
```

**Root cause**

VirtualBox had run an *unattended* installation. The evidence was an auto-generated
`Unattended-<guid>.iso` attached to SATA port 1 in place of the real ISO. Unattended setup selects
the first image in the installation file without asking, and the first image is Server Core, so
the Select Image screen never appeared at all.

The underlying mistake was a checkbox on the first page of the new-machine wizard. In VirtualBox
7.2 it reads **"Proceed with Unattended Installation"**, not "Skip Unattended Installation" as
older guides describe. The sense is inverted, so ticking it turns unattended installation **on**.

**Prevention**

Leave *Proceed with Unattended Installation* unticked. Confirm it took effect by expanding the
*Set up unattended guest OS installation* section: when unattended is off, its fields are greyed
out and cannot be edited. This was applied when building `WIN11-01`, and the manual installer
appeared as expected, which confirmed the diagnosis.

**Fix**

Replaced the unattended ISO with the real evaluation ISO under *Settings → Storage*, moved
*Optical* above *Hard Disk* in the boot order, booted from the ISO, deleted the three leftover
partitions, and selected *Windows Server 2025 Standard Evaluation (Desktop Experience)* manually
on the Select Image screen.

**What it taught**

Server Core cannot be converted to Desktop Experience after installation. That conversion path was
removed after Server 2012 R2. Choosing the wrong edition means a full reinstall, so the Select
Image screen is worth reading twice.

---

### Problem 3 — the machine booted the old install instead of the ISO

**Symptom**

The virtual machine went straight back into Server Core rather than starting Setup.

**Cause**

The five-second *Press any key to boot from CD or DVD* prompt was missed, and *Hard Disk* sat
above *Optical* in the boot order.

**Fix**

Moved *Optical* above *Hard Disk* under *Settings → System → Motherboard*. Pressing `F12`
during boot to reach the VirtualBox boot menu and choosing CD-ROM is more reliable than racing
the prompt.

---

### Building `WIN11-01`, the client

- Created from the Windows 11 Enterprise evaluation ISO: 4096 MB memory, 2 vCPU, 60 GB disk,
  adapter 1 on `LabNet`, adapter 2 on NAT.
- Enabled UEFI, Secure Boot and TPM 2.0 under *Settings → System → Motherboard*. Windows 11
  refuses to install without TPM 2.0 and Secure Boot, reporting *This PC cannot run Windows 11*.
- Left *Proceed with Unattended Installation* unticked, and manual Windows Setup appeared as
  intended.
- With UEFI enabled the legacy boot-order list is greyed out, because UEFI uses its own boot
  manager. When the machine dropped into the UEFI firmware menu instead of booting the ISO, the
  route through was *Boot Manager → UEFI VBOX CD-ROM*.
- At the account screen, chose *Sign-in options → Domain join instead* to create a local account
  rather than signing in with a Microsoft account. This machine joins the domain later, so a
  Microsoft account would only get in the way.

---

### Next (at the time)

- Install Guest Additions on `DC01` for display resizing, free mouse movement and clipboard sharing.
- Confirm the computer name is `DC01`.
- Identify which adapter is `LabNet` and set the static address `192.168.50.10` with the machine
  as its own DNS server. This must happen **before** promotion.
- Install the AD DS role and promote `DC01` to a domain controller for `kandel-lab.internal`.
  The Directory Services Restore Mode password is set at this step.
- Configure the DHCP scope `192.168.50.100`–`200`, on `LabNet` only.
- Activate the evaluation within ten days of installation and record the expiry date on the
  reference sheet.

---

## Session 2 — the domain comes up, and four faults that stopped it

**Goal:** get `kandel-lab.internal` actually serving a client, and join `WIN11-01` to it.

**Starting state:** `DC01` had been promoted and Active Directory was running, but nothing on
`192.168.50.0/24` worked. The client could not reach the domain controller at all.

Four faults, in the order they were found.

---

### Fault 1 — a domain user could not sign in at the domain controller console

**Symptom**

> The sign-in method you're trying to use isn't allowed. For more information, contact your
> network administrator.

A newly created domain user, signing in at `DC01` itself.

**First fix, which worked and was wrong**

Added the user to **Allow log on locally** in the Default Domain Controllers Policy and ran
`gpupdate /force`. The error went away. A second user hit the same thing, so **Domain Users**
went into the same policy.

**Root cause**

Domain controllers restrict interactive logon by default to Administrators, Account Operators,
Backup Operators, Print Operators and Server Operators. Ordinary users are excluded deliberately,
because a domain controller holds the entire directory.

The error was the security model working, not a misconfiguration.

**Correct fix**

Reverted the policy to its defaults. Ordinary users sign in to workstations, not to domain
controllers. Joined `WIN11-01` to the domain and tested logins there, where they work with no
policy change at all.

**What it taught**

The first fix that makes an error disappear is not always the right one. Granting the right
worked, and quietly reduced the security posture of the domain controller to do it.

---

### Fault 2 — the domain controller had no address on its own network

**Symptom**

Nothing on `192.168.50.0/24` responded. The client could not reach the domain controller.

**Diagnosis**

```powershell
Get-NetIPConfiguration
```

The LabNet adapter was on **169.254.82.13**, an APIPA address, with DNS set to `127.0.0.1`.
The NAT adapter held its own VirtualBox address with DNS pointing at the home router.

**Root cause**

The static address was never applied. `DC01` had been promoted while its lab adapter held no
valid address, so nothing existed on the lab subnet for anything to talk to.

**Fix**

Set `192.168.50.10 / 255.255.255.0`, no default gateway, DNS `127.0.0.1`, through `ncpa.cpl`.
Then re-registered the domain controller's records, which had been written against the APIPA
address:

```powershell
ipconfig /registerdns
```

followed by restarting **Netlogon** and **DNS Server**. Confirmed with:

```powershell
nslookup kandel-lab.internal
```

Also renamed both adapters to **LabNet** and **NAT**, so the two can never be confused again.

**What it taught**

When nothing on a subnet works, check whether anything is actually on that subnet before
blaming a service. A domain controller must hold a static address: on a lease it would register
DNS records that go stale, and clients would stop finding the domain.

---

### Fault 3 — the client was on a different network entirely

**Symptom**

`WIN11-01` had IP `10.0.2.15` and DNS `192.168.20.1`. A domain join was never going to work.

**Diagnosis**

```powershell
Get-NetAdapter
```

returned a single adapter, attached to NAT. There was no LabNet adapter at all.

**Root cause**

The virtual machine had been built with one NAT adapter. The two machines were on completely
separate networks, so no amount of DNS or DHCP troubleshooting would have helped.

**Fix**

Shut the machine down, changed Adapter 1 to Internal Network `LabNet`, and left Adapter 2
disabled through the domain join so that DNS could only come from the domain controller.

**What it taught**

Confirm the path exists before troubleshooting anything that runs over it. DNS cannot point at
the wrong server if there is no route to the right one either way.

---

### Fault 4 — DHCP was running and leasing nothing

**Symptom**

With the adapter corrected, the client still came up on **169.254.204.210**.

**Diagnosis**

Opened `dhcpmgmt.msc`. No scope existed, and the server node carried a red downward arrow.

**Root cause**

Two things at once. No scope had been created, and the server was not authorised in Active
Directory.

**Fix**

Created scope **LabNet**, `192.168.50.100–200 / 255.255.255.0`, with no default gateway, DNS
option `192.168.50.10`, and parent domain `kandel-lab.internal`. Authorised the server in AD and
confirmed the binding sat on LabNet only.

The client then leased correctly:

```
IPv4 Address . . . : 192.168.50.100
DHCP Server  . . . : 192.168.50.10
DNS Servers  . . . : 192.168.50.10
DNS Suffix   . . . : kandel-lab.internal
Default Gateway  . :
```

**What it taught**

In a domain, a DHCP server must be authorised in Active Directory before it will answer
anything. The service runs, the scope reports active, and the network stays silent, which looks
exactly like a dead network. The red arrow on the server node is the tell. The mechanism exists
so that a rogue DHCP server cannot be plugged into a corporate network and hand out bad
addresses.

---

### What was completed

- `DC01` on static `192.168.50.10`, DNS records re-registered against the correct address
- DHCP installed, scope created, authorised in Active Directory, bound to LabNet only
- Organisational unit structure created: `KANDEL-LAB` containing `Users`, `Computers` and `Groups`
- Users created inside the correct OU rather than the default Users container, which cannot have
  Group Policy linked to it
- `WIN11-01` renamed, joined to `kandel-lab.internal`, and its computer object moved into
  `KANDEL-LAB\Computers`
- Domain user sign-in verified on the client, with no Group Policy modification required
- Client leasing `192.168.50.100` with DNS `192.168.50.10` and the correct DNS suffix

### Next (at the time)

- Security groups, and three to five Group Policy Objects
- VirtualBox snapshots of both machines
- Guest Additions on `DC01` for clipboard sharing
- Reduce `DC01` to 2 GB RAM now that promotion is complete

---

## Session 3 — groups, Group Policy, and a setting that went nowhere

**Goal:** give the domain some rules. Build a directory structure that looks like a real small
business, then apply Group Policy to it and prove the policies actually landed.

**Starting state:** domain live, client joined, DHCP working. No groups, no policies, no structure
beyond the three top-level OUs.

---

### The scenario

Building against a made-up business rather than clicking settings at random, so that every policy
has a reason behind it.

**Kandel Marine Services.** Small Hobart business, 25 staff, one office, one domain.

| Department | Needs |
|---|---|
| Management | Most access, few restrictions |
| Office | Shared drive, locked-down desktop, no software installs |
| Workshop | Shared computer on a floor. Heavily restricted, short screen lock |
| IT | Full control, exempt from the restrictions |

---

### Directory structure

```
KANDEL-LAB
├── Users
│   ├── Management
│   ├── Office
│   ├── Workshop
│   └── IT
├── Computers
└── Groups
```

**OUs are not prefixed, groups are.** An OU's full path already identifies it
(`KANDEL-LAB\Users\Workshop`), so a prefix adds nothing. Group names are flat and have to be
unique across the whole domain, and they appear mixed in with built-in groups in every permission
dialog in Windows. A prefix makes them sort together and makes it obvious at a glance which ones
are mine.

All groups use the company prefix `GK-`, Global scope, Security type.

| Group | Members |
|---|---|
| `GK-Management` | pkandel, mmaraseni |
| `GK-Office` | mdmaraseni |
| `GK-Workshop` | sdahal |
| `GK-IT` | gkandel |
| `GK-All-Staff` | all five |

---

### Naming convention, and the collision

Users are `first initial + surname`. Two of the five broke it immediately:

- Manisha Maraseni → `mmaraseni`
- Madan Maraseni → also `mmaraseni`

Resolved by adding the second letter of the first name: `mdmaraseni`.

Trivial in a five-user lab, but every real organisation hits this, usually in the first week. The
point is to decide the rule before it happens rather than improvising per user and ending up with
three conventions in the same directory.

**Lab shortcut, deliberate:** accounts were created with a shared initial password and
*User must change password at next logon* ticked, then changed at first sign-in. That mirrors real
onboarding. The initial password is recorded outside this repository.

---

### GPO 1 — password and lockout policy

Linked at the **domain root**, not an OU.

| Setting | Value |
|---|---|
| Minimum password length | 12 |
| Password complexity | Enabled |
| Minimum password age | 1 day |
| Maximum password age | 90 days |
| Account lockout threshold | 5 attempts |
| Account lockout duration | 15 minutes |
| Reset counter after | 15 minutes |

**The thing worth knowing:** password and account lockout settings only take effect when applied at
the domain root. Link the same GPO to an OU and the settings save, report as applied, and do
nothing at all. No error, no warning. Password policy is a property of the domain itself, not of
the users inside it, so there is one per domain.

**`Allow Administrator account lockout` set to Disabled.** This is a deliberate lab decision. The
domain administrator is the only administrative account here, and locking it out would mean
recovering through Directory Services Restore Mode. In production this would be Enabled, with a
documented break-glass account and a recovery process.

#### Verification

Signed in as `sdahal` on `WIN11-01` and entered the wrong password six times:

> The referenced account is currently locked out and may not be logged on to.

Confirmed from the server:

```powershell
Search-ADAccount -LockedOut
Unlock-ADAccount -Identity sdahal
```

Three things proven in one test: the policy applied from the domain root, the threshold of 5 is
real, and the account genuinely locks and can be released.

---

### GPO 2 — Workshop lockdown

Linked to the `Workshop` OU.

| Setting | Half of the GPO |
|---|---|
| Enable screen saver | User Configuration |
| Screen saver timeout, 300 seconds | User Configuration |
| Password protect the screen saver | User Configuration |
| Prohibit access to Control Panel and PC settings | User Configuration |
| All Removable Storage classes: Deny all access | **Computer Configuration** |

#### Verification

```
gpresult /r
```

```
USER SETTINGS
    CN=Sanjiv Dahal,OU=Workshop,OU=Users,OU=KANDEL-LAB,DC=kandel-lab,DC=internal
    Group Policy was applied from: DC01.kandel-lab.internal

    Applied Group Policy Objects
    -----------------------------
        GK - Workshop Lockdown
```

Control Panel returned the expected restriction message. Targeting by OU works.

---

### Fault 5 — the same folder name in both halves of the editor

**Symptom**

The screen saver settings described in every guide were not in Control Panel → Personalization.
The folder was there, but it contained lock screen settings instead: *Prevent lock screen
background motion*, *Force a specific default lock screen and logon image*.

**Root cause**

A Group Policy object has two independent halves, **Computer Configuration** and **User
Configuration**, and several folder names appear in both with completely different contents.
`Control Panel → Personalization` exists in both. The computer half holds lock screen settings.
The user half holds screen saver settings.

**Fix**

Scrolled to the top of the editor's left pane and worked down the correct root node.

**What it taught**

The path alone is not enough to identify a setting. `Control Panel → Personalization` is ambiguous
and half the guides on the internet omit which root they mean. Read the top of the tree before
trusting the folder.

---

### Fault 6 — a blocked setting that reports the wrong error

**Symptom**

Right-click desktop → Personalise produced:

> ms-settings:personalization
> This file does not have an app associated with it for performing this action. Please install an
> app or, if one is already installed, create an association in the Default Apps Settings page.

Which reads like a broken file association, not a policy.

**Root cause**

*Prohibit access to Control Panel and PC settings* blocks the `ms-settings:` URI handler. Windows
tries to launch `ms-settings:personalization`, finds the handler blocked, and falls back to a
generic "no associated app" message instead of reporting a restriction.

**Confirmation**

Control Panel itself, launched directly, returned the correct message:

> This operation has been cancelled due to restrictions in effect on this computer.

So the policy was working. Only the error text was misleading.

**What it taught**

This is a help desk ticket waiting to happen. The user reports that personalisation is broken and
asks what app they need to install. The real cause is a deliberate policy applied by someone else,
possibly years earlier. Without knowing that, an hour disappears into file associations and
default apps, chasing a problem that does not exist.

---

### Fault 7 — the USB restriction went nowhere

**Symptom**

*All Removable Storage classes: Deny all access* was Enabled in a GPO that `gpresult` confirmed had
applied. Removable storage was not blocked.

**Diagnosis**

```
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\RemovableStorageDevices
```

```
ERROR: The system was unable to find the specified registry key or value.
```

The key the policy would create does not exist, so the setting never reached the machine.

**Root cause**

The GPO is linked to the `Workshop` OU, which contains **user** objects. `WIN11-01` lives in
`KANDEL-LAB\Computers`.

Computer Configuration settings apply to computer objects inside the linked OU. They have nothing
to do with who is signed in. Sanjiv being in `Workshop` does not place the machine he is sitting at
in `Workshop`. The user half of the GPO applied correctly, the computer half had no computers to
apply to, and Windows reported no error because nothing had gone wrong from its point of view.

**Fix, two options**

1. Create `KANDEL-LAB\Computers\Workshop`, move `WIN11-01` into it, and link the GPO there as well.
2. Enable **loopback processing**, which makes the user half of a GPO follow the computer rather
   than the user.

Option 2 is the better fit for the scenario. A workshop machine should be locked down because of
*where it is*, not because of who happens to be standing at it. That is the same reason it is used
for kiosks, shared terminals and library computers. To be implemented next session.

**What it taught**

A GPO reporting as applied does not mean every setting inside it did anything. `gpresult /r` shows
which GPOs were processed, not which settings took effect. The registry is the place to confirm a
specific setting landed, and the absence of a key is a clear answer rather than a dead end.

The wider lesson: Computer Configuration and User Configuration are targeted independently. Putting
both kinds of setting in one GPO linked to one OU will quietly work for half of them.

---

### What was completed

- Snapshots of both machines before any changes
- `DC01` reduced from 4 GB to 2 GB RAM now that promotion is complete
- Four department OUs under `KANDEL-LAB\Users`
- Five users with a documented naming convention, including a resolved collision
- Five security groups, Global scope, company prefix
- Domain-level password and lockout policy, tested to lockout and recovered with PowerShell
- Workshop lockdown GPO, applied and verified with `gpresult`
- Three faults documented, one of which is still open by design
- Guest Additions installed on both machines, with the shared clipboard set to bidirectional

### Next

- Loopback processing on the Workshop GPO, so the restrictions follow the machine
- Drive mapping for the Office OU, which needs a file share on `DC01` first
- Company wallpaper, same dependency
- Security filtering to exempt `GK-IT` from the restrictions
