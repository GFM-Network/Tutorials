## Backup Before Attempting This!

### Step 1: Delete Old Partitions and Create New Ones

1. Boot into your partition manager of choice (e.g., Parted Magic, [download link](https://partedmagic.com/)).
2. Use GParted to:
   - Delete extra partitions, including the old FAT32 system partition (these should be small, between 100 MB and 2 GB).
   - Create a new NTFS partition with 512 MB of space and label it "System Reserved".
3. Shutdown.

### Step 2: Setup Partitions with Correct Drive Letters

1. Launch Windows 10 Install disc/USB.
2. Press Shift+F10 once you see the language select screen to bring up the command prompt.
3. Launch DISKPART:
   - Select the volume with 512 MB (e.g., `select vol 0`).
   - Assign a letter (e.g., `G:` or any available letter except `C:`).
   - Exit.
   - If Windows operating system is NOT on C:
     - Select the volume with the Windows installation.
     - Assign letter `C:`.
     - Exit.

Example output:
diskpart
select disk 0
list vol
select vol 0 <---500 MB Partition
assign letter=G:
select vol 1 <--- Large Partition with Windows Install on it
assign letter=C:
exit

### Step 3: Copy Boot Records, Fix MBR, and Rebuild BCD

1. Rebuild the new system reserved partition with EFI and regular boot files:
bcdboot c:\Windows /s <boot letter of System Reserved Partition>: /f ALL
bcdboot c:\Windows /s G: /f ALL

2. Rebuild the BCD and master boot record:
- Verify the system sees the Windows installation by running:
  ```
  bootrec /scanos <--- Verify it sees the Windows install
  ```
- Run the following commands to perform the repairs:
  ```
  bootrec /fixmbr
  bootrec /fixboot <---- Pick the installation will mirror /scanos bootrec /rebuildbcd
  ```

[Watch a video tutorial](https://www.youtube.com/watch?v=F72D7uL6cZg).