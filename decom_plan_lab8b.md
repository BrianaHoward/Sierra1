# WINTHIRTYTWO — Lab 8 Part B Decomissioning Plan (Final)
**Version:** 1.2 — Review draft only — **not executed**.
**Target:** WINTHIRTYTWO (Windows Server 2025, Domain Controller, Hyper-V lab host)
**Owner:** Briana Howard | **Repo:** Sierra1

> Scope: lab-only cleanup. Preserved: **Completed Part A backups on Secondary (E:)** and the **Sierra1 GitHub repository**. Everything else lab-created is removed to satisfy the "no remnants" requirement.

## 0. Pre-flight Probe (one short log line each)

| # | Probe | Pass criterion | Stop action |
|---|-------|----------------|-------------|
| 0.1 | Confirm role + domain state | `Get-CimInstance Win32_ComputerSystem \| Select DomainRole` → `4`. `Test-Connection WINTHIRTYTWO` works | Resolve DC reachability before touching AD. |
| 0.2 | Confirm VMs present | `Get-VM` lists the lab VMs and their VHD/X files | Fix Hyper-V reachability. |
| 0.3 | Record current state | Write a short baseline to `decom_run.log` (VM list, features, AD state) | Create log dir if missing. |

## 1. Decommission and Delete All Lab Virtual Machines + VM Files

| Step | Action | Command | Expected result |
|------|--------|---------|-----------------|
| 1.1 | Stop each lab VM | `Stop-VM -Name <vm> -Force` | All lab VMs shut down. |
| 1.2 | Delete each lab VM | `Remove-VM -Name <vm> -Force` | VM removed from Hyper-V. |
| 1.3 | Delete VM disks/temp files | `Remove-Item -Path <vm>.vhdx, <vm>.log, <vm>.avhdy` -Force | No lab VHD/VHDX, logs, or temp files remain. |
| 1.4 | Verify | `Get-VM; Get-ChildItem C:\ -Filter *.vhd* -ErrorAction SilentlyContinue` | No lab VM remains. |

**Pass criterion:** No lab VM, no VM disk file, no VM log/temp file on the host.

## 2. Remove Applications and Files Created/Installed for Lab Work

| Step | Action | Command | Expected result |
|------|--------|---------|-----------------|
| 2.1 | Remove lab-installed apps/packages | `Get-Package -Name <labapp> -ProviderName MSI \| Uninstall-Package -Name <labapp> -ProviderName MSI` | Lab apps removed. |
| 2.2 | Remove lab-installed Windows features | `Get-WindowsFeature -Name <labfeat> \| Remove-WindowsFeature -Name <labfeat>` | Lab features removed. |
| 2.3 | Remove lab service | `Stop-Service -Name <labsvc> -Force`; `Set-Service -Name <labsvc> -StartupType Disabled` | Service stopped and disabled. |
| 2.4 | Remove lab files/folders | `Remove-Item -Path <labdir> -Recurse -Force` | Lab file/folder removed. |
| 2.5 | Verify | `Get-ChildItem C:\ -Directory -Recurse \| Where {$_.Name -like '*lab*'}` | No lab artifacts. |

**Preserved (never deleted):** Completed Part A backups on **Secondary (E:)** and the **Sierra1 GitHub repository**.

**Pass criterion:** No lab application, lab service, lab feature, or lab-created file/folder remains.

## 3. Demote WINTHIRTYTWO as Domain Controller and Remove AD DS Configuration

| Step | Action | Command | Expected result |
|------|--------|---------|-----------------|
| 3.1 | Identify FSMO owners | `Get-ADDomain \| Select PDCEmulator, RIDMaster, PrimaryDC, SchemaMaster` | FSMO roles identified. |
| 3.2 | Demote the DC | Run on WINTHIRTYTWO as the local Domain Controller:
`Uninstall-ADDSDomainController -Force -DemoteOperationMasterRole -LastDomainControllerInDomain` | Server demoted to standalone. |
| 3.3 | Remove AD DS role and configuration | `Remove-WindowsFeature -Name AD-Domain-Services -Remove` (only if still present on the server) | AD DS role/configuration removed. |
| 3.4 | Verify | `netdom query fsmo` → empty; `Get-WindowsFeature -Name AD-Domain-Services` → disabled/absent | WINTHIRTYTWO is no longer a Domain Controller. |

> `Remove-ADDSDomain` is omitted unless a forest-level domain removal is actually required; it is not part of this lab's demotion scope.

**Pass criterion:** WINTHIRTYTWO is no longer a Domain Controller, no longer trusted as a DC, and the AD DS role/configuration has been removed from the server.

## 4. Verify No Lab-Related Remnants Remain

| Step | Action | Command | Expected result |
|------|--------|---------|-----------------|
| 4.1 | Re-check VMs | `Get-VM` | Empty. |
| 4.2 | Re-check lab files/features/services | `Get-ChildItem C:\ -Directory -Recurse \| Where {$_.Name -like '*lab*'}`; `Get-WindowsFeature`; `Get-Service` | None. |
| 4.3 | Re-check AD role | `Test-Connection WINTHIRTYTWO`; `Get-WindowsFeature -Name AD-Domain-Services` | Server standalone; AD feature disabled/absent. |
| 4.4 | Re-run probes 0.1–0.3 | Same as Step 0 | All green. |

**Pass criterion (ALL must hold):** No VMs, no lab files/applications/services, no Domain Controller role, no AD configuration for WINTHIRTYTWO remains.

## 5. Output

- Save this plan as `decom_plan_lab8b.md` in the Sierra1 repo folder for upload.
- After execution, a companion `decom_run.log` will record: which steps completed, which did not, and the next probe at each point.
