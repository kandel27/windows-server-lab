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

### Next

- Install Guest Additions on `DC01` for display resizing, free mouse movement and clipboard sharing.
- Confirm the computer name is `DC01`.
- Identify which adapter is `LabNet` and set the static address `192.168.50.10` with the machine
  as its own DNS server. This must happen **before** promotion.
- Install the AD DS role and promote `DC01` to a domain controller for `kandel-lab.internal`.
  The Directory Services Restore Mode password is set at this step.
- Configure the DHCP scope `192.168.50.100`–`200`, on `LabNet` only.
- Activate the evaluation within ten days of installation and record the expiry date on the
  reference sheet.
