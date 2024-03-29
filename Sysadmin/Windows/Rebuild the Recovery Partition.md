## Setup Recovery Partition

1. **Create Recovery Partition**: Allocate at least 1GB of space for the recovery partition.
2. **Format Partition**: Format the recovery partition as NTFS.

## DISKPART Changes

- Select disk 0.
- Select the last partition (partition 3).
- Display partition details.
- Set the partition ID to "de94bba4-06d1-4d40-a16a-bfd50179d6ac" (GPTDisk).

    - **Note**: For MBR, use id=27; for GPT, use id="ebd0a0a2-b9e5-4433-87c0-68b6b72699c7".
    - _Create Partition if Missing!_
        ```plaintext
        cre par pri size=605 id=de94bba4-06d1-4d40-a16a-bfd50179d6ac
        ```
    - Format the partition as NTFS quickly with the label "WinRE".
        ```plaintext
        format fs=ntfs quick label=WinRE
        ```
    - Assign drive letter "Z".
        ```plaintext
        assign letter=z
        ```
    - For UEFI only, set attributes.
        ```plaintext
        gpt attributes=0x8000000000000001
        ```

## Configure WinRE

- Disable WinRE:
    ```plaintext
    reagentc /disable
    ```
- Verify presence of WinRE.wim:
    ```plaintext
    ls "C:\Windows\System32\Recovery" | FindStr /I "WinRE.wim" ; ls "C:\Windows\System32\Recovery" -Hidden | FindStr /I "WinRE.wim"
    ```
    - If missing, copy WinRE.wim and ReAgent.xml from installation media to `C:\Windows\System32\Recovery`.

- Enable WinRE:
    ```plaintext
    reagentc /enable
    ```
    - Verify settings:
    ```plaintext
    reagentc /info
    ```

- If `reagentc /enable` fails:
    - Manually copy WinRE.wim to `Z:\Recovery\WindowsRE`.
    - Set the REimage path:
        ```plaintext
        reagentc /SetREimage /Path "Z:\Recovery\WindowsRE"
        ```
    - Enable WinRE:
        ```plaintext
        reagentc /Enable
        ```
    - Verify with:
        ```plaintext
        reagentc /Info
        ```

## Rebooting to Recovery Partition

- Boot into the recovery menu by holding Shift while restarting.
- Alternatively, allow the boot process to fail three times.

## Recovery Tools and Options

- **Startup Repair**: Occasionally useful for resolving boot issues.
- **Startup Settings**: Provides access to safe mode.
- **Command Prompt**: Allows various recovery tasks like file copying, disk checking, and system file repairs.
- **Regedit**: For registry editing.
- **Uninstall Updates**: Revert problematic updates.
- **UEFI Firmware Settings**: Access BIOS settings.
- **System Restore and System Image Recovery**: Last resort options for system recovery.

[Watch a video tutorial](https://www.youtube.com/watch?v=L7Ss-KKp010).

## Reagentc /enable failing with error code 3bc3, Windows Update failing with error 800F0922
If you are having similar issues, you can check whether this is the problem with the diskpart command. Assuming you have only one hard disk drive,

```cmd
select disk 0
list partition

You should see something like this:

Partition ###  Type              Size     Offset
-------------  ----------------  -------  -------
Partition 1    System             500 MB  1024 KB
Partition 2    Primary            445 GB   501 MB

The smaller partition named "System" is the one you want, usually partition 1, so

select partition 1
detail partition

And you should see something like this:

Partition 1
Type    : c12a7328-f81f-11d2-ba4b-00a0c93ec93b
Hidden  : Yes
Required: No
Attrib  : 0XC000000000000000
Offset in Bytes: 1048576

Volume ###  Ltr  Label        Fs     Type        Size     Status     Info
----------  ---  -----------  -----  ----------  -------  ---------  --------
Volume 3                      FAT32  Partition    500 MB  Healthy    System

If the file system is not FAT32 then you are not looking at the right partition. It should also be Hidden, and will not usually have a drive letter assigned unless (as in the originally posted question) it has been explicitly given one for troubleshooting purposes. It might not be exactly 500MB, but should only be taking up a small fraction of the hard disk.

The type of the EFI partition should be c12a7328-f81f-11d2-ba4b-00a0c93ec93b as shown above. If it is not, and in particular if it is ebd0a0a2-b9e5-4433-87c0-68b6b72699c7 (see Microsoft Basic Data Partition on Wikipedia) then that is likely to be the cause of the problem.

If the partition type is incorrect, you can fix this with the set id command,

set id=c12a7328-f81f-11d2-ba4b-00a0c93ec93b

The reagentc /enable command should then work, and if you are lucky, so will Windows Update.