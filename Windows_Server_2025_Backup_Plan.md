WINDOWS SERVER 2025 BACKUP PLAN (FINAL — WITH DOMAINROLE CORRECTION)
=================================================================

Scope: Files, Active Directory, Group Policy only.
Destination for all backups: Secondary (E:) drive.
No backup actions have been executed. This is the final plan for your review.

------------------------------------------------------------------
PART 1 - IDENTIFY WHAT TO BACK UP  (you run these read-only checks)
------------------------------------------------------------------

The plan must not assume a C:\Data folder. First identify the real
folders and drives in your environment.

Step 1 - List all drives/volumes:
  Open PowerShell and run:
    Get-Volume
  Note the DriveLetter, FileSystem, Size, and Health for every volume.

Step 2 - List the top-level folders on each data volume:
  For each volume that is NOT the backup target (i.e. not E:), run:
    Get-ChildItem -Path <DriveLetter>:\ -Directory
  Example, on D:  Get-ChildItem -Path D:\ -Directory

Step 3 - Find large folders, to spot what normally consumes the most space:
  For each data volume:
    Get-ChildItem -Path <DriveLetter>:\ -Directory |
      Sort-Object { (Get-ChildItem -Path $_.FullName -Recurse -ErrorAction SilentlyContinue |
                     Measure-Object -Property Length -Sum).Sum } -Descending |
      Select-Object -First 20 -Property Name,@{Name='SizeGB';Expression={[math]::Round(($_.Name | Get-ChildItem -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum).Sum / 1GB,2)}}

Step 4 - If this server is a file server / file share host:
  PowerShell:
    Get-FsrmQuota
    Get-FsrmShare
  Or open Server Manager > File and Storage Services.

Step 5 - Report back to me the results of Steps 1-4:
  - All drive letters and their roles (system, data, backup target).
  - The top-level folders you want backed up.
  - Any folders you deliberately want to exclude.

Do not run any backup command until you have reported the results of
Steps 1-5 and I have confirmed the final list.

------------------------------------------------------------------
PART 2 - ACTIVE DIRECTORY BACKUP (dedicated System State command)
------------------------------------------------------------------

Use the dedicated Windows Server 2025 System State backup command:

    wbadmin start systemstatebackup -backupTarget:E: -quiet

What System State includes:
  - Active Directory database and transaction logs
  - SYSVOL (the files GPOs use)
  - Boot sector, registry, COM+ class registration database, and
    other critical system state data that AD depends on.

Step A - Confirm this server is a Domain Controller:
  Run in an elevated Command Prompt or PowerShell:
    (Get-CimInstance Win32_ComputerSystem).DomainRole

  Interpretation of the result (per Microsoft documentation):
    0 = Standalone workstation
    1 = Member workstation
    2 = Standalone server
    3 = Member server
    4 = Backup domain controller
    5 = Primary domain controller

  If the value is 4 or 5, this server is a Domain Controller.

  Quick secondary check (also a definitive indicator of a DC):
    Get-Service NTDS -ErrorAction SilentlyContinue
  This returns the Active Directory Domain Services service only on a
  Domain Controller.

  Notes on the command:
    - This is the dedicated Windows Server 2025 System State backup
      command. It does not use `-include:C:`; it backs up the local
      computer's System State directly.
    - `-backupTarget:E:` sends the backup to the Secondary (E:) drive.
    - `-quiet` suppresses the progress UI so the command can run
      unattended in Task Scheduler.
    - You must run this on a Domain Controller (it backs up the local
      computer's System State).
    - `-allCritical` is NOT used here. It is not required for a System
      State backup. -allCritical is only needed if you want every
      critical volume included.

Step B - Run the backup:
  Open an elevated Command Prompt and run:
    wbadmin start systemstatebackup -backupTarget:E: -quiet

Step C - Wait for the job to finish:
  The command prints a progress line. It finishes successfully when
  the line reads: "The backup operation completed successfully."
  A timestamped backup folder is created under E: (look for
  Windows/SystemBackup or a date-time folder, depending on the
  version of Windows Server Backup).

Step D - Verify the backup is readable:
  Open the backup folder under E: and confirm it contains a SystemState
  subfolder with adls, edb.chk, ntds.dit, etc.

------------------------------------------------------------------
PART 3 - GROUP POLICY BACKUP (GPMC method)
------------------------------------------------------------------

Use the Group Policy Management Console (GPMC) backup. This is the
proper method for backing up GPOs, and is more complete than copying
SYSVOL.

Step A - Open GPMC:
  Press Win+R, type:  gpmc.msc
  (You can also run it from an elevated Command Prompt.)

Step B - Back up the GPOs:
  In the left pane, right-click the domain you want to back up, and
  click BACK UP. (You can also right-click a single GPO.)
  Choose the backup location:  E:\GPO_Backup\GPMC
  (Create the folder E:\GPO_Backup\GPMC first.)
  Click OK and wait for the backup to finish.

Step C - What the GPMC backup includes:
  - The GPO definitions (XML metadata for each GPO)
  - The registry-based policy files (the .pol DWORD settings)
  - A GPMC-managed backup history

Step D - Relationship to SYSVOL and the System State backup:
  - The AD System State backup in Part 2 already includes SYSVOL.
  - SYSVOL holds the actual GPO files (scripts, reports, and the
    machine/user policy data).
  - If you ALSO want a pure file-level copy of SYSVOL for the GPOs,
    run:
      robocopy C:\Windows\SYSVOL E:\GPO_Backup\Sysvol /MIR /R:3 /W:5
    This is optional and is in addition to the GPMC backup, not a
    replacement for it.

Step E - Verify:
  Open E:\GPO_Backup\GPMC and confirm you see a folder for each
  backed-up domain, containing a "GPOs" subfolder with backed-up
  GPOs (.xml files).

------------------------------------------------------------------
PART 4 - FILE BACKUP (after Part 1)
------------------------------------------------------------------

Use one of these methods for only the folders you identified in Part 1.
Do NOT back up the entire C: drive.

Method A - robocopy (best for specific folders):
  robocopy <source> <E:\Backup\<name>> /MIR /R:3 /W:5 /LOG:E:\Backup\robocopy.log

Method B - Windows Server Backup for entire data volumes:
  wbadmin start backup -backupTarget:E: -include:<DriveLetter> -quiet

  Example, backing up only the D: drive:
    wbadmin start backup -backupTarget:E: -include:D: -quiet

  Notes:
    - Replace <DriveLetter> with each data drive you want to back up.
    - Do not include C: unless you specifically want a full system
      backup, and do not use -allCritical.
    - Do not back up E: itself; it is the destination.

Step 1 - After you report the results of Part 1, tell me which
folders and drives you selected. I will then confirm the exact
commands to run, in order.

------------------------------------------------------------------
PART 5 - VERIFICATION (after each run)
------------------------------------------------------------------
  - Confirm E:\Backup\ (or E:\Files\) exists with the files.
  - Confirm the AD System State backup contains SystemState data.
  - Confirm E:\GPO_Backup\GPMC contains backed-up GPOs.
  - At least once, test that you can restore a file, an AD component,
    and a GPO from E:.

------------------------------------------------------------------
PART 6 - SCHEDULING / AUTOMATION
------------------------------------------------------------------
  - Open Task Scheduler (taskschd.msc) and create a new Task.
  - Action: powershell.exe, or a saved .ps1 script on E: containing
    the commands from Parts 2-4.
  - Check the options:
      - Run whether user is logged in or not
      - Run with highest privileges
      - Stop the task if it fails
  - Store all logs under E:\Backup\
  - Do not include the entire C: drive in a scheduled backup. Include
    only the data drives or folders you identified in Part 1.

------------------------------------------------------------------
NOTES
------------------------------------------------------------------
  - No backup actions were executed. This plan is for your review only.
  - You asked to avoid the entire C: drive and -allCritical. This
    plan respects that. -allCritical is not used anywhere in this plan.
    -include:C: is no longer used because the Active Directory backup
    in Part 2 now uses the dedicated System State command
    `wbadmin start systemstatebackup`, which backs up the local
    computer's System State (including AD, SYSVOL, and the boot
    sector) without needing a manual -include drive specification.
  - If you have multiple Domain Controllers, you may repeat the System
    State backup on each one. A consistent snapshot of one DC is
    normally enough.
  - Do NOT include E: itself in any backup, because it is the storage
    destination.

------------------------------------------------------------------
NEXT STEP
------------------------------------------------------------------
  1. Run the Part 1 read-only commands above and tell me what you
     find (drive letters, roles, and the folders you want backed up).
  2. I will then confirm the final file/folder list and give you the
     exact commands to run, in order. No backup actions will be run
     until you approve them.
