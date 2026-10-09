
# Windows Bootloader, EFI/System Partition & WinRE Recovery

A comprehensive guide to repairing corrupted Windows 10/11 installations, rebuilding EFI/System partitions, restoring Windows boot files, repairing system corruption, and configuring the Windows Recovery Environment (WinRE).

This guide is intended for system administrators working from Windows installation media using **Shift + F10**.

> [!WARNING]
> Back up important data before modifying partitions.
>
> This procedure preserves the existing Windows installation. **Do not delete or format the main Windows partition.**
>
> Deleting every partition or running `diskpart clean` will destroy the existing partition layout and may erase Windows, applications, and personal files.

---

## Table of Contents

1. [Preparation](#1-preparation)
2. [Identify Disks and Partitions](#2-identify-disks-and-partitions)
3. [Determine GPT or MBR](#3-determine-gpt-or-mbr)
4. [Rebuild EFI Partition (GPT/UEFI)](#4-rebuild-efi-partition-gptuefi)
5. [Rebuild System Partition (MBR/BIOS)](#5-rebuild-system-partition-mbrbios)
6. [Repair Windows Installation Corruption](#6-repair-windows-installation-corruption)
7. [Rebuild Windows Recovery Partition](#7-rebuild-windows-recovery-partition)
8. [Restore WinRE.wim](#8-restore-winrewim)
9. [Configure Windows Recovery](#9-configure-windows-recovery)
10. [Post-Recovery Verification](#10-post-recovery-verification)
11. [Troubleshooting](#11-troubleshooting)
12. [Real-World Recovery Case](#12-real-world-recovery-case)
13. [References](#13-references)

---

# 1. Preparation

## Requirements

- Windows 10/11 installation USB or ISO
- Administrator access
- Access to BIOS/UEFI settings
- Sufficient free space for missing system partitions
- Backup of important data

### Boot into Windows PE

1. Boot from the Windows installation USB.
2. Wait for the Windows Setup screen.
3. Press **Shift + F10**.
4. Command Prompt will open.

Example:

```cmd
X:\Sources>
```

`X:` represents the temporary Windows Preinstallation Environment (WinPE).

It is not your installed Windows partition.

### Example Drive Letters

Throughout this guide, the following letters are used:

| Letter | Purpose |
|---|---|
| D: | Installed Windows |
| S: | EFI/System partition |
| R: | Windows Recovery partition |
| X: | Windows PE environment |

> [!IMPORTANT]
> Drive letters may change between Windows PE and normal Windows.
>
> Always verify the correct letters before executing commands.

---

# 2. Identify Disks and Partitions

Start DiskPart:

```cmd
diskpart
```

List physical disks:

```cmd
list disk
```

Example:

```text
Disk ###  Status         Size     Free     Gpt
--------  -------------  -------  -------  ---
Disk 0    Online          238 GB      0 B   *
Disk 1    Online          931 GB   1001 MB  *
```

Select the Windows disk:

```cmd
select disk 1
```

Display partitions:

```cmd
list partition
```

Display volumes:

```cmd
list volume
```

Inspect the selected disk:

```cmd
detail disk
```

Exit DiskPart:

```cmd
exit
```

### Locate the Windows Installation

```cmd
dir C:\Windows
dir D:\Windows
dir E:\Windows
```

Verify essential files:

```cmd
dir D:\Windows\System32\winload.efi
dir D:\Windows\System32\Config\SYSTEM
```

For this guide, assume the installed Windows is on `D:`.

---

# 3. Determine GPT or MBR

Run:

```cmd
diskpart
list disk
```

Check the `Gpt` column.

| Disk type | Indicator | Boot mode |
|---|---|---|
| GPT | `*` under GPT | Usually UEFI |
| MBR | No `*` | Usually Legacy BIOS |

Also verify the actual firmware boot mode.

### Partition Types

| Partition | GPT/UEFI | MBR/BIOS |
|---|---|---|
| Boot/System | FAT32 EFI | NTFS System Reserved |
| Windows | NTFS | NTFS |
| Recovery | NTFS | NTFS |
| MSR | Usually 16 MB | Not applicable |

### Important Partition IDs

**GPT EFI System Partition**

```text
c12a7328-f81f-11d2-ba4b-00a0c93ec93b
```

**GPT Windows Recovery Partition**

```text
de94bba4-06d1-4d40-a16a-bfd50179d6ac
```

**GPT Microsoft Basic Data**

```text
ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
```

**MBR Windows Recovery Partition**

```text
27
```

> [!CAUTION]
> Do not confuse the EFI System Partition with the Windows Recovery partition.
>
> They serve completely different purposes.

---

# 4. Rebuild EFI Partition (GPT/UEFI)

Use this section for Windows installed in UEFI mode on a GPT disk.

## Step 1 — Check for an Existing EFI Partition

```cmd
diskpart
select disk 1
list partition
list volume
```

Look for a small FAT32 EFI System Partition.

Typical size:

- 100 MB
- 260 MB
- 300 MB or larger, depending on configuration

If a healthy EFI partition already exists, preserve it and assign an available letter.

For example, after identifying its volume number:

```cmd
select volume 2
detail volume
assign letter=S
exit
```

Proceed to Step 3.

## Step 2 — Create a New EFI Partition

Only perform this step if the target GPT disk has sufficient **contiguous unallocated space** and requires a new EFI partition.

Recommended example size: 260 MB.

```cmd
diskpart
select disk 1
create partition efi size=260
format quick fs=fat32 label="System"
assign letter=S
list partition
exit
```

Expected result:

```text
DiskPart succeeded in creating the specified partition.
```

> [!WARNING]
> Never format an existing partition unless you have verified that it contains no data you need.
>
> If insufficient space is available, stop and assess whether the Windows partition can be safely shrunk.

## Step 3 — Rebuild Windows Boot Files

Verify the Windows installation:

```cmd
dir D:\Windows
```

Recreate boot files:

```cmd
bcdboot D:\Windows /s S: /f UEFI /v
```

Expected result:

```text
Boot files successfully created.
```

This command recreates the Windows boot files and BCD store on the selected EFI partition.

> [!NOTE]
> Using `/s` with BCDBoot does not create a new UEFI firmware NVRAM boot entry.
>
> Additional firmware boot configuration may be necessary.

## Step 4 — Verify the BCD Store

```cmd
dir S:\EFI\Microsoft\Boot\BCD /a
```

Inspect the boot configuration:

```cmd
bcdedit /store S:\EFI\Microsoft\Boot\BCD /enum all
```

Typical Windows Boot Loader:

```text
Windows Boot Loader
-------------------
identifier              {default}
device                  partition=D:
path                    \Windows\system32\winload.efi
osdevice                partition=D:
systemroot              \Windows
```

Identifiers and drive letters may differ.

## Step 5 — Verify Firmware Boot Order

Restart into BIOS/UEFI.

Confirm:

1. UEFI boot mode is enabled.
2. The correct Windows Boot Manager is selected.
3. The intended Windows disk is being used.
4. Other Windows installations and their boot entries remain intact.

If Windows still boots into recovery, continue to Section 6.

---

# 5. Rebuild System Partition (MBR/BIOS)

> [!IMPORTANT]
> This section applies only to Legacy BIOS installations using MBR disks.
>
> Do not run these commands on GPT/UEFI installations.

## Step 1 — Locate the System Reserved Partition

```cmd
diskpart
select disk 0
list partition
list volume
```

Look for the NTFS System Reserved partition.

If it exists, preserve it and assign an unused letter.

If it is missing and sufficient contiguous unallocated space is available, create a new partition:

```cmd
create partition primary size=550
format quick fs=ntfs label="System Reserved"
assign letter=S
active
exit
```

The `active` command marks the selected MBR partition as the BIOS boot partition.

## Step 2 — Rebuild Boot Files

```cmd
bcdboot D:\Windows /s S: /f BIOS /v
```

Expected result:

```text
Boot files successfully created.
```

## Step 3 — Repair MBR Boot Code If Necessary

When an MBR boot-code problem is suspected:

```cmd
bootrec /fixmbr
```

Then, if appropriate:

```cmd
bootrec /fixboot
```

If `/fixboot` returns:

```text
Access is denied.
```

Do not repeatedly force it.

Verify the correct active partition and BCDBoot configuration.

Additional diagnostic commands:

```cmd
bootrec /scanos
bootrec /rebuildbcd
```

> [!NOTE]
> `bootrec /rebuildbcd` is not normally necessary after a successful BCDBoot rebuild.
>
> These commands do not repair corrupted Windows operating system files.

---

# 6. Repair Windows Installation Corruption

This is an important section.

Even when EFI and BCD have been rebuilt successfully, Windows may still fail to start because of corrupted operating system files.

Common symptoms:

- Windows boots directly into Troubleshoot.
- "Choose your keyboard layout" appears repeatedly.
- Startup Repair fails.
- BCDBoot succeeds but Windows does not start.
- Windows repeatedly enters automatic recovery.

## Step 1 — Verify Essential Files

```cmd
dir D:\Windows\System32\winload.efi
dir D:\Windows\System32\Config\SYSTEM
```

If the files exist, continue.

## Step 2 — Check Windows Component Store

Create a writable scratch directory:

```cmd
mkdir D:\DISMScratch
```

Check for component-store corruption:

```cmd
dism /image:D:\ /cleanup-image /checkhealth /scratchdir:D:\DISMScratch
```

Possible result:

```text
The component store is repairable.
The operation completed successfully.
```

This indicates that corruption was detected and is considered repairable.

For a more comprehensive scan:

```cmd
dism /image:D:\ /cleanup-image /scanhealth /scratchdir:D:\DISMScratch
```

> [!NOTE]
> `/CheckHealth` and `/ScanHealth` are diagnostic operations. They do not repair corrupted files.

## Step 3 — Repair Windows Using Offline SFC

Execute:

```cmd
sfc /scannow /offbootdir=D:\ /offwindir=D:\Windows
```

Wait for completion.

Successful repair output:

```text
Windows Resource Protection found corrupt files and
successfully repaired them.
```

**This was the command that resolved the Windows boot failure in the documented recovery case.**

After SFC finishes:

1. Close Command Prompt.
2. Remove the installation USB.
3. Restart the computer.
4. Select the correct Windows Boot Manager.
5. Test Windows startup.

Do not modify additional partitions if Windows now boots correctly.

## Step 4 — Repair the Component Store If Necessary

If Windows remains damaged, DISM can attempt a repair:

```cmd
dism /image:D:\ /cleanup-image /restorehealth /scratchdir:D:\DISMScratch
```

A compatible repair source may be required when working offline.

For example, after identifying a suitable Windows installation WIM and its matching index:

```cmd
dism /image:D:\ /cleanup-image /restorehealth /source:wim:E:\sources\install.wim:INDEX /limitaccess /scratchdir:D:\DISMScratch
```

Replace `INDEX` with the appropriate image index.

The repair source should match the installed Windows architecture, edition, and servicing requirements.

If the installation media contains `install.esd`, use the appropriate ESD source syntax or prepare a suitable WIM.

After a successful DISM repair, run offline SFC again.

---

# 7. Rebuild Windows Recovery Partition

Windows Recovery Environment (WinRE) provides:

- Startup Repair
- Safe Mode / Startup Settings
- Command Prompt
- System Restore
- Uninstall Updates
- System Image Recovery
- Other recovery tools

WinRE is separate from the EFI System Partition.

## Step 1 — Check Existing Recovery Configuration

If Windows boots normally, open Command Prompt as Administrator:

```cmd
reagentc /info
```

Expected result:

```text
Windows RE status: Enabled
```

If WinRE is already working, do not recreate its partition unnecessarily.

## Step 2 — Create a Recovery Partition

Boot from Windows installation media and press Shift + F10.

Start DiskPart:

```cmd
diskpart
select disk 1
list partition
```

Only create a partition if no suitable recovery partition exists.

For this example, allocate 1536 MB of contiguous unallocated space:

```cmd
create partition primary size=1536
format quick fs=ntfs label="WinRE"
assign letter=R
exit
```

Create the WinRE directory:

```cmd
mkdir R:\Recovery\WindowsRE
```

> [!WARNING]
> Do not delete or format an existing recovery partition without verifying its contents and purpose.

---

# 8. Restore WinRE.wim

The Windows Recovery Environment requires the `Winre.wim` image.

## Step 1 — Check the Windows Installation

```cmd
dir D:\Windows\System32\Recovery /a
```

Look for:

```text
Winre.wim
```

Check the recovery partition:

```cmd
dir R:\Recovery\WindowsRE /a
```

If the image already exists in the recovery partition, it may be reusable.

## Step 2 — Copy WinRE.wim

If `Winre.wim` exists in the Windows installation:

```cmd
copy D:\Windows\System32\Recovery\Winre.wim R:\Recovery\WindowsRE\
```

Verify:

```cmd
dir R:\Recovery\WindowsRE /a
```

## Step 3 — Extract WinRE.wim From Windows Installation Media

If the image is missing from both locations, check the installer USB:

```cmd
dir E:\sources\install.*
dir F:\sources\install.*
```

Look for:

```text
install.wim
```

or:

```text
install.esd
```

### For install.wim

Inspect the image:

```cmd
dism /Get-WimInfo /WimFile:E:\sources\install.wim
```

Identify the correct Windows edition and index.

Create a temporary mount directory on a writable volume with sufficient free space:

```cmd
mkdir D:\Mount
```

Mount the image read-only:

```cmd
dism /Mount-Image /ImageFile:E:\sources\install.wim /Index:INDEX /MountDir:D:\Mount /ReadOnly
```

Replace `INDEX` with the correct number.

Check for WinRE:

```cmd
dir D:\Mount\Windows\System32\Recovery /a
```

Copy the recovery image:

```cmd
copy D:\Mount\Windows\System32\Recovery\Winre.wim R:\Recovery\WindowsRE\
```

Unmount the image:

```cmd
dism /Unmount-Image /MountDir:D:\Mount /Discard
```

> [!NOTE]
> Some Windows installation media use `install.esd`, split WIM files, or compressed images.
>
> These may require exporting or preparing a suitable image before mounting.
>
> Use a compatible WinRE image for the installed Windows version.

---

# 9. Configure Windows Recovery

## Step 1 — Register WinRE From Windows PE

Assuming:

- Windows installation: `D:\Windows`
- Recovery image: `R:\Recovery\WindowsRE`

Run:

```cmd
reagentc /setreimage /path R:\Recovery\WindowsRE /target D:\Windows
```

This registers the recovery image for the offline Windows installation.

## Step 2 — Configure GPT Recovery Partition Attributes

Return to DiskPart:

```cmd
diskpart
select disk 1
list partition
```

Identify the correct WinRE partition number.

For example:

```cmd
select partition 3
detail partition
```

**Only after verifying the correct recovery partition**, set its type:

```cmd
set id=de94bba4-06d1-4d40-a16a-bfd50179d6ac
```

Set GPT attributes:

```cmd
gpt attributes=0x8000000000000001
```

Exit:

```cmd
exit
```

These settings identify the partition as Windows Recovery and prevent normal automatic drive-letter assignment.

### MBR Recovery Partition

For MBR systems, the recovery partition uses:

```cmd
set id=27
```

Do not use GPT attributes on MBR disks.

## Step 3 — Enable WinRE

Boot into the installed Windows.

Open Command Prompt as Administrator:

```cmd
reagentc /enable
```

Verify:

```cmd
reagentc /info
```

Expected:

```text
Windows RE status: Enabled
```

Confirm that the recovery location references the intended partition.

Remove any temporary recovery drive letter after verification if it remains assigned.

---

# 10. Post-Recovery Verification

Once Windows boots successfully, complete the following checks.

## Step 1 — Repair Windows Component Store

Open Command Prompt as Administrator:

```cmd
DISM /Online /Cleanup-Image /RestoreHealth
```

Wait for completion.

## Step 2 — Verify Windows System Files

```cmd
sfc /scannow
```

Ideal result:

```text
Windows Resource Protection did not find any integrity violations.
```

## Step 3 — Verify Boot Configuration

```cmd
bcdedit /enum {current}
```

Check the active Windows boot loader.

## Step 4 — Verify WinRE

```cmd
reagentc /info
```

Expected:

```text
Windows RE status: Enabled
```

## Step 5 — Test Startup

Restart Windows several times.

Confirm:

- Windows boots normally.
- The intended Windows Boot Manager is used.
- No automatic recovery loop occurs.
- Windows system files are healthy.
- WinRE is available.
- Other installed Windows systems remain bootable.

> [!IMPORTANT]
> On computers with multiple drives, do not remove an old EFI partition until the target Windows installation has been verified to boot independently.

---

# 11. Troubleshooting

| Problem | Possible cause | Recommended action |
|---|---|---|
| `bootrec /fixboot` Access is denied | EFI/boot configuration issue | Verify boot mode and use BCDBoot |
| `bootrec /rebuildbcd` finds no installation | Incorrect partition or boot configuration | Verify Windows location |
| BCDBoot succeeds but Windows enters Troubleshoot | Windows corruption or wrong boot entry | Inspect BCD and run offline SFC |
| BCD store cannot be opened | Wrong EFI drive letter or missing BCD | Verify the EFI partition |
| `Winre.wim` missing | Recovery image unavailable | Locate or extract compatible image |
| DISM reports repairable corruption | Component-store corruption | Repair with suitable source |
| SFC repairs corrupt files | Damaged Windows files | Restart and test boot |
| `reagentc /enable` fails | Invalid WinRE configuration | Verify image path and partition type |
| Windows boots but WinRE is disabled | Recovery not configured | Register and enable WinRE |

## Common Error: Access Is Denied

Command:

```cmd
bootrec /fixboot
```

Error:

```text
Access is denied.
```

For UEFI/GPT installations, use the correct EFI partition and rebuild Windows boot files:

```cmd
bcdboot D:\Windows /s S: /f UEFI
```

## Common Error: BCD Store Not Found

Command:

```cmd
bcdedit /store S:\EFI\Microsoft\Boot\BCD /enum
```

Error:

```text
The boot configuration data store could not be opened.
The system cannot find the file specified.
```

Verify:

```cmd
diskpart
list volume
exit
```

Then:

```cmd
dir S:\EFI\Microsoft\Boot /a
```

If the directory is missing, confirm that `S:` is the correct EFI partition before rebuilding anything.

## Common Problem: Windows Boots Into Troubleshoot

Symptoms:

```text
Choose your keyboard layout
Troubleshoot
Advanced options
```

If the correct bootloader is present and Windows is still failing, investigate Windows corruption.

Run:

```cmd
dism /image:D:\ /cleanup-image /checkhealth
```

Then:

```cmd
sfc /scannow /offbootdir=D:\ /offwindir=D:\Windows
```

Restart after successful repairs.

---

# 12. Real-World Recovery Case

## Original Disk Configuration

A Windows computer had two GPT disks:

| Disk | Size | Description |
|---|---|---|
| Disk 0 | 238 GB | Separate Windows installation with EFI partition |
| Disk 1 | 931 GB | Target Windows installation |

The target Windows installation on Disk 1 did not have its own EFI System Partition.

## Initial Symptoms

- Windows failed to boot normally.
- `bootrec /fixboot` returned Access is denied.
- The computer repeatedly entered the Windows Recovery screen.
- BCDBoot successfully created boot files, but Windows still failed to start.

## Recovery Procedure

### 1. Create an EFI Partition on Disk 1

```cmd
diskpart
select disk 1
create partition efi size=260
format quick fs=fat32 label="System"
assign letter=S
exit
```

### 2. Rebuild Windows Boot Files

```cmd
bcdboot D:\Windows /s S: /f UEFI /v
```

Result:

```text
Boot files successfully created.
```

However, Windows still entered recovery.

### 3. Check Windows Corruption

```cmd
dism /image:D:\ /cleanup-image /checkhealth
```

Result:

```text
The component store is repairable.
The operation completed successfully.
```

### 4. Run Offline SFC

```cmd
sfc /scannow /offbootdir=D:\ /offwindir=D:\Windows
```

Result:

```text
Windows Resource Protection found corrupt files and
successfully repaired them.
```

### 5. Restart Windows

After restarting, Windows booted successfully.

## Root Cause and Lessons Learned

The repair confirmed that Windows system-file corruption was present and that SFC successfully repaired corrupted files.

The successful startup occurred after both the EFI boot files had been rebuilt and SFC had repaired the installation.

The precise original cause of the boot failure was not independently established, but the case demonstrates that bootloader repair alone may be insufficient.

### Key Lessons

1. A successful BCDBoot operation does not guarantee Windows itself is healthy.
2. Do not assume `bootrec /fixboot` is necessary on UEFI/GPT systems.
3. Always identify the correct disk and Windows partition.
4. Repair Windows system-file corruption before repeatedly rebuilding boot partitions.
5. Keep EFI and WinRE partitions separate.
6. Preserve existing Windows installations and other disks whenever possible.
7. Use offline SFC when Windows cannot boot normally.

---

# 13. References

## Microsoft Documentation

- [BCDBoot Command-Line Options](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdboot-command-line-options-techref-di?view=windows-11)
- [Windows System and Recovery Partitions](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/capture-and-apply-windows-system-and-recovery-partitions?view=windows-11)
- [REAgentC Command-Line Options](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/reagentc-command-line-options?view=windows-11)
- [Repair a Windows Image](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/repair-a-windows-image?view=windows-11)

## Original GFM-Network Tutorials

- [Rebuild a New Boot Partition](https://github.com/GFM-Network/Tutorials/blob/main/Sysadmin/Windows/Rebuild%20a%20new%20boot%20partition.md)
- [Rebuild the Recovery Partition](https://github.com/GFM-Network/Tutorials/blob/main/Sysadmin/Windows/Rebuild%20the%20Recovery%20Partition.md)

---

**GFM-Network — Windows System Administration Tutorials**
