---
title: Windows Sandbox Dual-GPU and 0x80070002 Recovery Notes
description: Debugging notes for an intermittent Windows Sandbox disconnect and a later 0x80070002 initialization failure fixed by removing and reinstalling the Sandbox feature.
tags:
- windows
- windows-sandbox
- hyper-v
- rdp
- gpu
- debugging
---

## Purpose

- This note records a collaborative debugging session with Claude and Codex.
- It covers two related Windows Sandbox failures:
  - an intermittent RDP disconnect associated with the mixed AMD and NVIDIA GPU environment
  - a later initialization failure with error `0x80070002`
- The first problem improved after disabling the AMD integrated GPU.
- About one day later, Sandbox stopped initializing entirely.
- The later failure may have been related to repeatedly disabling and enabling the integrated GPU, but this was not proven.
- The final recovery was to remove Windows Sandbox, reset its generated container data, and reinstall the optional feature.
- This note separates:
  - confirmed observations
  - hypotheses
  - rejected explanations
  - unverified advice
  - the successful recovery flow

## Initial Symptom

- Windows Sandbox opened a window with a black display area.
- A loading dialog then appeared.
- After a short wait, Sandbox displayed a message similar to:

```txt
The connection to Windows Sandbox was lost.
Do you want to reconnect?
```

- Selecting reconnect repeated the same sequence.
- The behavior was random:
  - some launches failed
  - some launches became stable
  - changing the `.wsb` configuration did not produce a reliable result

## System Environment

- Windows build:
  - Windows 11 25H2
  - build `26200.8655`
- CPU:
  - AMD Ryzen 7 7700
- GPUs:
  - NVIDIA GeForce RTX 4070
  - AMD Radeon integrated graphics from the Ryzen 7 7700
- Memory:
  - 32 GB RAM
- Virtualization environment:
  - WSL 2 worked normally
  - `VirtualMachinePlatform` was enabled
  - Hyper-V services were running
- Previous software:
  - VMware Workstation had been installed and removed
  - several VMware services and virtual network adapters remained
- Windows Sandbox package observed during the investigation:
  - version `0.5.3.0`

## First Diagnostic Questions

- The initial investigation considered:
  - disabled virtualization
  - WSL and Hyper-V interaction
  - VMware remnants
  - a damaged Hyper-V virtual switch
  - incomplete Windows updates
  - RDP policy restrictions
  - missing RDP display components
  - a Sandbox saved-state problem
  - a host and guest component version mismatch
  - a mixed-GPU driver conflict

## Basic Virtualization Checks

- WSL reported:

```txt
Default Version: 2
```

- `VirtualMachinePlatform` reported:

```txt
State: Enabled
```

- This showed that the base virtualization platform was working.
- A working WSL 2 installation did not prove that every Sandbox display component was healthy.
- However, it made a total Hyper-V failure unlikely.

## VMware Remnants

- The system still had running VMware services:

```txt
VMAuthdService
VMnetDHCP
VMUSBArbService
VMware NAT Service
```

- It also still had VMware virtual network adapters:

```txt
VMware Network Adapter VMnet1
VMware Network Adapter VMnet8
```

- The Hyper-V default switch was present and active.
- Claude initially treated the VMware remnants as the root cause.
- The VMware services were stopped and disabled.
- The VMware virtual adapters were also disabled or removed during testing.
- Windows Sandbox continued to fail.

## VMware Conclusion

- VMware remnants were real and worth cleaning.
- They were not sufficient to explain this failure.
- Disabling them did not change the core symptom.
- The investigation therefore moved away from networking and VMware services as the primary cause.

## Hyper-V and RDP Log Clues

- Hyper-V logged:

```txt
Virtual machine requested unsupported Virtual PCI protocol version 0x10006
```

- The request returned:

```txt
0x8007051A
Indicates two revision levels are incompatible.
```

- The VM then successfully negotiated:

```txt
Virtual PCI protocol version 0x10005
```

- Terminal Services also logged:

```txt
An error occurred when transitioning from DesktopLocked
in response to EvDesktopLocked.
ErrorCode 0x8007139F
```

## Version-Mismatch Hypothesis

- Claude initially interpreted the VPCI fallback as evidence of:
  - an incomplete Windows update
  - mismatched host and guest Hyper-V components
  - a damaged Sandbox base image
- This was plausible because the guest requested `0x10006` while the host negotiated `0x10005`.
- Later cross-run comparison showed the same VPCI warning during successful launches.
- Therefore:
  - the warning might indicate version skew
  - it was not the direct trigger for the random disconnect

## Windows Component Health

- DISM was executed:

```powershell
DISM /Online /Cleanup-Image /RestoreHealth
```

- The result was:

```txt
Image Version: 10.0.26200.8655
The restore operation completed successfully.
The operation completed successfully.
```

- The component store did not report corruption.
- Repeating DISM or SFC was therefore unlikely to solve the intermittent behavior.
- The apparently empty `OsRevision` field from one `Get-ComputerInfo` query was not proof of a broken update.
- DISM showed the complete build and revision as `26200.8655`.

## RDP Policy Hypothesis

- The host registry showed:

```txt
fDenyTSConnections = 1
fSingleSessionPerUser = 1
```

- Claude proposed that `fDenyTSConnections = 1` was blocking Sandbox's internal RDP connection.
- It was changed to `0`.
- Sandbox failed in the same way afterward.
- Port `3389` was listening.
- No decisive policy override was found.

## RDP Policy Conclusion

- Changing `fDenyTSConnections` did not fix the problem.
- The setting was not the root cause.
- The conversation's claim that Sandbox necessarily required this host setting to be `0` was not confirmed by the test.
- Registry changes that alter host RDP policy should not be used casually as a Sandbox repair.

## Sandbox Process State

- During a failed launch, these processes existed:

```txt
vmmemWindowsSandbox
vmwp
WindowsSandboxRemoteSession
WindowsSandboxServer
```

- `WindowsSandboxRemoteSession` reported:
  - `Responding = True`
  - window title `Windows Sandbox`
- No matching Application log crash was found.
- The process remained alive even after the visible Sandbox window disappeared and only the reconnect dialog remained.
- Reconnecting opened the larger window again, showed a loading dialog, and then returned to the disconnected message.

## What the Process State Proved

- The Sandbox VM was not simply failing before startup.
- The VM worker, memory process, server process, and remote-session client all existed.
- The failure happened later in the display or session path.
- This shifted attention from:
  - VM creation
  - storage creation
  - basic networking
- Toward:
  - RDP session initialization
  - virtual display initialization
  - guest-side session shutdown

## Display Configuration Tests

- Three launch configurations were compared.

### Configuration A

- Short description:
  - vGPU off
  - network on

```xml
<Configuration>
  <VGpu>Disable</VGpu>
</Configuration>
```

### Configuration B

- Short description:
  - vGPU off
  - network off
  - redirections off

```xml
<Configuration>
  <VGpu>Disable</VGpu>
  <MemoryInMB>4096</MemoryInMB>
  <AudioInput>Disable</AudioInput>
  <VideoInput>Disable</VideoInput>
  <ClipboardRedirection>Disable</ClipboardRedirection>
  <PrinterRedirection>Disable</PrinterRedirection>
  <Networking>Disable</Networking>
</Configuration>
```

### Vanilla

- Short description:
  - default settings
  - vGPU on
  - network on

## Early vGPU Observations

- Disabling vGPU changed the size of the black display area.
- This showed that the rendering path changed.
- It did not reliably prevent the disconnect.
- The minimal configuration also failed.
- Therefore:
  - vGPU settings affected presentation
  - vGPU settings alone did not explain the failure
  - networking and redirection were not required to reproduce it

## Fully Classified Launch Window

- Codex reconstructed a classifiable launch window from `12:12:46` to `12:30:05`.
- A correction was made:
  - the `12:20:27` run was Vanilla, not A

| Configuration | Stable | Failed | Total |
|---|---:|---:|---:|
| A | 1 | 5 | 6 |
| B | 3 | 4 | 7 |
| Vanilla | 0 | 1 | 1 |

## Transition Results

- The destination launch determines whether the transition is marked stable or failed.

| Transition | Stable | Failed | Total |
|---|---:|---:|---:|
| A to A | 0 | 2 | 2 |
| A to B | 1 | 1 | 2 |
| A to Vanilla | 0 | 1 | 1 |
| B to A | 1 | 2 | 3 |
| B to B | 1 | 3 | 4 |
| B to Vanilla | 0 | 0 | 0 |
| Vanilla to A | 0 | 1 | 1 |
| Vanilla to B | 0 | 0 | 0 |
| Vanilla to Vanilla | 0 | 0 | 0 |

## Exact Classified Sequence

```txt
B(S) -> B(F) -> B(S) -> A(F) -> B(F) -> B(F) -> A(F)
-> Vanilla(F) -> A(F) -> A(F) -> A(F) -> B(S) -> B(F) -> A(S)
```

- `S` means stable.
- `F` means failed.
- Older Jump List records showed:
  - 16 total A activations
  - 8 total B activations
- The older records were excluded because they could not reliably distinguish:
  - Vanilla launches
  - explicit launches
  - automatic reconnect attempts

## Cross-Run Interpretation

- Repeating the same configuration did not guarantee the same result.
- Switching between A and B did not guarantee success.
- B could succeed and then fail.
- A could fail repeatedly and later succeed.
- The sample was small, so no transition rule was statistically conclusive.
- The data did establish that:
  - configuration order was not a reliable predictor
  - networking was not the primary cause
  - redirection was not the primary cause
  - `.wsb` configuration alone could not explain the randomness

## Timing Observation

- `WindowsSandboxServer` remained alive after a Sandbox window closed.
- In one observation:

```txt
12:24:06 - WindowsSandboxServer still present
12:24:36 - WindowsSandboxServer no longer present
```

- This led to a cooldown or server-initialization hypothesis.
- Waiting sometimes appeared to improve the chance of success.
- Repeated classified runs did not reveal a reliable warm-up sequence.
- Timing probably influenced the race, but waiting was not a dependable repair.

## Codex Log Correlation

- Codex compared retained logs across stable and failed runs.
- The strongest common sequence was:
  - VM restoration completed
  - storage setup completed
  - account and registry setup completed
  - networking setup completed when enabled
  - RDP connected
  - graphics initialized
  - login succeeded
  - the server or guest terminated the session shortly afterward

- Failed sessions were disconnected about:
  - minimum: `1.4` seconds after login
  - maximum: `3.1` seconds after login
  - median: `1.95` seconds after login

- The RDP disconnect reason was:

```txt
Reason 3
```

- This indicated a server-initiated disconnect.

## vGPU Log Evidence

- vGPU creation and opening completed in both stable and failed runs.
- The driver logs reported:

```txt
STATUS_SUCCESS
DXGK_VGPU_FAILURE_NONE
```

- No corresponding evidence was found for:
  - a host GPU reset
  - an application crash
  - a Hyper-V worker crash
  - a network failure

- This was important:
  - the vGPU log did not show a direct GPU operation failure
  - the problem occurred after successful graphics and RDP initialization

## Saved-State Hypothesis

- Every retained launch restored the same Sandbox saved-state snapshot.
- The initial Codex inference was a race during:
  - saved-state resume
  - guest desktop initialization
  - RDP session initialization

- Sandbox state files were found:

```txt
C:\ProgramData\Microsoft\Windows\Containers\Dumps\...\*.vmrs
C:\ProgramData\Microsoft\Windows\Containers\Snapshots\...\SnapshotSavedState.vmrs
```

- The saved-state and dump files were deleted to force regeneration.
- The random disconnect remained.

## Saved-State Conclusion

- Saved-state restoration may have influenced timing.
- The specific saved-state files were not the root cause.
- Deleting them did not solve the issue.
- The earlier statement that one snapshot file was definitely the problem was disproved by the next test.

## Missing `rdpudd` and `RdpIdd` Hypothesis

- File checks did not find:

```txt
C:\Windows\System32\drivers\rdpudd.sys
C:\Windows\System32\rdpudd.dll
```

- This led Claude to declare that the RDP display driver was missing.
- A Windows 11 24H2 ISO was mounted and searched.
- The ISO did not contain those files at the assumed paths.
- `rdpidd.inf` and an `RdpIdd.dll` package were later found in DriverStore.
- `pnputil` was used to add `rdpidd.inf`.
- `RdpIdd.dll` was also copied manually to `System32`.
- These actions did not establish a reliable fix.

## Why the Missing-File Theory Was Weak

- The system was Windows 11 25H2, but the searched ISO was 24H2.
- Modern display-driver packaging does not necessarily match the assumed old file and service layout.
- The RDP log had already reported:

```txt
The listener listens with display driver rdpudd.dll available.
```

- Most importantly:
  - the problem was intermittent
  - a truly required, permanently missing file would normally produce a consistent failure
  - A and B still sometimes worked

- The user explicitly challenged this contradiction.
- Claude then revised the theory from a missing file to a timing race.

## Earlier Feature Reinstallation Test

- During the original random-disconnect investigation, Sandbox was disabled and immediately enabled again with dependencies:

```powershell
Disable-WindowsOptionalFeature `
  -Online `
  -FeatureName Containers-DisposableClientVM `
  -NoRestart

Enable-WindowsOptionalFeature `
  -Online `
  -FeatureName Containers-DisposableClientVM `
  -All `
  -NoRestart
```

- After a restart:
  - the first launch failed
  - later attempts succeeded
  - the failure returned the next day

- This temporary improvement did not prove that a missing component had been restored.
- It was consistent with a state or timing change caused by reinstalling and restarting.
- This was not the same as the later successful `0x80070002` recovery:
  - there was no restart between disabling and enabling the feature
  - the generated container data was not reset
  - the full Sandbox image state was therefore not regenerated

## Repair-Upgrade Theory

- Claude repeatedly proposed an in-place Windows repair upgrade.
- The proposed purpose was to refresh:
  - Sandbox components
  - Hyper-V components
  - Features on Demand
  - the guest image

- A 24H2 ISO could not be used for an in-place repair of a 25H2 installation.
- A matching 25H2 image would have been required.
- The repair upgrade was never shown to be necessary because the later GPU isolation test fixed the problem.

## Most Useful Isolation Test

- Codex recommended testing without the AMD integrated GPU:
  - connect displays to the RTX 4070
  - disable `AMD Radeon(TM) Graphics` in Device Manager
  - restart
  - repeat Sandbox launches

- The AMD iGPU was disabled.
- Windows Sandbox then worked.
- This was the decisive experiment.

## What Was Actually Confirmed

- Confirmed:
  - the system had both AMD and NVIDIA display adapters
  - the failure was intermittent while both adapters were enabled
  - A, B, and Vanilla could all fail
  - networking and redirection did not determine the result
  - the VM, RDP connection, graphics initialization, and login could complete before failure
  - the guest or server initiated the disconnect about two seconds after login
  - disabling the AMD iGPU removed the observed failure

- Not directly proven:
  - that Hyper-V randomly selected one GPU by name
  - that the AMD driver alone was defective
  - that the NVIDIA driver was completely uninvolved
  - that VPCI fallback caused the failure
  - that a specific RDP display file was missing
  - that the Sandbox saved-state snapshot was corrupt

## Best Root-Cause Statement

- The strongest supported conclusion is:
  - the failure was caused by an unstable interaction in the mixed AMD iGPU and NVIDIA dGPU display virtualization path

- The likely affected area was:
  - Windows Sandbox virtual display initialization
  - Hyper-V graphics virtualization
  - the RDP display path
  - AMD display-driver state

- A reasonable inferred sequence is:

```txt
Sandbox restores or starts the VM
-> virtual devices initialize
-> RDP connects
-> graphics and login complete
-> mixed-GPU display state becomes unhealthy
-> Sandbox guest or server closes the RDP session
```

- The exact internal Windows component that made the final decision was not identified.

## Why the Failure Looked Random

- Both GPUs were available to the Windows graphics stack.
- The launch involved multiple asynchronous components:
  - Sandbox server
  - Hyper-V VM worker
  - virtual PCI devices
  - graphics virtualization
  - RDP transport
  - guest desktop initialization
- A mixed-GPU driver state could change with:
  - boot state
  - sleep and resume
  - previous Sandbox launches
  - driver initialization timing
  - device enable and disable operations
- This explains why configuration changes appeared to help occasionally without producing a repeatable rule.

## Claude and Codex Contributions

- Claude:
  - guided the long interactive troubleshooting session
  - proposed commands for Hyper-V, RDP, DISM, drivers, Sandbox state, and Windows features
  - helped eliminate several possible causes
  - eventually connected the problem to the dual-GPU environment
  - also made several premature root-cause declarations

- Codex:
  - reconstructed and classified repeated launches
  - corrected a misclassified Vanilla launch
  - compared stable and failed event sequences
  - showed that RDP and graphics initialization succeeded before disconnect
  - showed that VPCI fallback occurred in successful runs too
  - ruled out networking and `.wsb` redirection settings as primary causes
  - recommended the AMD iGPU isolation test
  - interpreted the later AMD driver unload event

- The collaboration worked best when:
  - Claude's broad hypothesis generation was checked against Codex's cross-run evidence
  - a hardware isolation test replaced further speculation

## Diagnostic Mistakes

- Several conclusions were stated too confidently before being tested:
  - VMware remnants were declared the root cause
  - VPCI version mismatch was declared the root cause
  - `fDenyTSConnections` was declared the root cause
  - a saved-state file was declared the root cause
  - missing `rdpudd` files were declared the root cause
  - a repair upgrade was described as the only solution

- Each theory was weakened or rejected by later evidence.
- The main reasoning error was anchoring:
  - once the investigation focused on missing display components, later clues were interpreted through that theory
  - the intermittent nature of the problem should have triggered an earlier reassessment

## Important Epistemic Lesson

- A log warning is not automatically the failure cause.
- Compare it across:
  - successful runs
  - failed runs
- If the same warning occurs in both, it is probably:
  - background noise
  - a tolerated fallback
  - a contributing condition rather than the trigger

- A fix is not confirmed by one successful run.
- For intermittent problems:
  - repeat the same test many times
  - classify every run
  - record the exact configuration
  - record whether the previous run was closed cleanly
  - compare event timing

## GPU Workaround for the Original Disconnect

- If the AMD iGPU is not needed:
  - connect every monitor to the RTX 4070
  - enter the motherboard BIOS
  - set the initial or primary display to PCIe
  - disable integrated graphics
  - save and restart

- This prevents Windows Sandbox from enumerating the AMD iGPU.
- The motherboard HDMI and DisplayPort outputs will no longer work while the iGPU is disabled.
- The RTX 4070 remains available for:
  - display output
  - 3D acceleration
  - video encoding
- This fixed the original random disconnect during the test period.
- It should not be described as a permanent repair because a different Sandbox initialization failure appeared about one day later.

## Keeping the AMD iGPU Enabled

- The conversation's proposed maintenance path was:
  - back up the BitLocker recovery key
  - suspend BitLocker before a firmware update
  - update the motherboard BIOS
  - update the AMD chipset driver
  - clean-install an appropriate AMD integrated graphics driver
  - restart
  - enable the iGPU again
  - repeat Sandbox launch testing

- Codex reported that the installed BIOS and AMD graphics driver were old at the time of diagnosis.
- Specific "latest" version numbers from the conversation should be rechecked before downloading anything.
- Keeping the iGPU enabled was not confirmed to be stable during this conversation.

## Temporary Workaround

- In Device Manager:
  - disable `AMD Radeon(TM) Graphics`
  - perform a full restart
  - launch Sandbox with only the NVIDIA GPU active

- This was the workaround that actually fixed the original disconnect behavior.

## Unverified `PreferredPhysicalGpu` Registry Advice

- Claude suggested:

```powershell
$regPath = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Virtualization"
Set-ItemProperty `
  -Path $regPath `
  -Name "PreferredPhysicalGpu" `
  -Value "NVIDIA GeForce RTX 4070"
```

- The value was written successfully.
- It did not solve the problem.
- Claude then suggested using the NVIDIA device instance ID instead of the display name.
- Claude later acknowledged that there was no verified evidence that Hyper-V read this value.
- The advice was speculative and internally inconsistent.

## Registry Cleanup

- If the unverified value was created, remove only that value:

```powershell
Remove-ItemProperty `
  -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Virtualization" `
  -Name "PreferredPhysicalGpu" `
  -ErrorAction SilentlyContinue
```

- Do not delete the entire `Virtualization` registry key.
- The key contains legitimate Hyper-V configuration values.

## Later `System.Exception` Dialog

- After the AMD iGPU had been disabled, Sandbox later displayed:

```txt
Exception of type System.Exception was thrown
```

- This dialog was generic and did not identify the real cause.
- The nearby system log showed:

```txt
\Driver\amduw23g failed to unload
```

- `amduw23g` was associated with the AMD graphics driver.
- The event happened shortly after waking from sleep.
- Device Manager showed the AMD adapter as disabled.

## Interpretation of the Later Exception

- The AMD driver was likely left in a partially unloaded state after:
  - sleep or resume
  - disabling the device
  - changing GPU state without a full restart

- Sandbox VM disk creation still succeeded.
- Startup then aborted and the Sandbox client reduced the underlying failure to a generic `System.Exception`.
- The immediate recovery was:
  - perform a full Windows restart
  - test Sandbox again

- To avoid recurrence:
  - do not toggle the iGPU while Sandbox is active
  - restart after enabling or disabling a display adapter
  - prefer disabling the iGPU in BIOS if it will remain unused

## New Failure After One Day

- About one day after the original GPU workaround, Windows Sandbox developed a new failure.
- The message was:

```txt
Windows Sandbox failed to initialize.
The system cannot find the file specified.
Error 0x80070002
```

- This failure differed from the original issue:
  - the original VM could start and then lose its RDP session
  - the new failure occurred while Sandbox was being initialized
- The failure happened after the integrated GPU had been disabled and enabled multiple times.
- That timing makes the device-state changes a possible contributor.
- However, no log proved that GPU toggling directly corrupted the Sandbox image or container cache.

## Initial 0x80070002 Diagnosis

- Windows Sandbox itself was present and correctly signed.
- The investigation found component-servicing errors including:
  - `CorruptPayloadFile`
  - `STATUS_SXS_FILE_HASH_MISMATCH`
  - a failed repair involving Host Networking Service files
  - `FIOReadFileIntoBuffer` returning `0x80070002`
- Sandbox Server could start, but `StartSandbox` returned `0x80070002`.
- Plain Sandbox and custom `.wsb` configurations failed in the same way.
- This pointed toward an incomplete or inconsistent Sandbox payload rather than a `.wsb` configuration problem.

## DISM and SFC Repair Attempt

- The following commands were run from an elevated terminal:

```powershell
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow
```

- DISM reported:

```txt
The restore operation completed successfully.
The operation completed successfully.
```

- SFC reported that Windows Resource Protection found no integrity violations.
- Windows was restarted and Sandbox was tested again.
- Error `0x80070002` still occurred.
- This established that:
  - Windows component-store repair completed
  - ordinary protected system files passed verification
  - the active problem remained in Sandbox-specific generated state or feature installation

## Refined Diagnosis

- After DISM and SFC:
  - the Sandbox application activated
  - parent VHD files existed
  - linkage identifiers matched
  - a new Sandbox child VHD could be created
  - image preparation still returned `0x80070002`
- Hyper-V and Host Networking Service did not receive a complete VM launch request.
- The most useful working diagnosis was:
  - the generated Sandbox image or container cache was incomplete
  - or the current Sandbox feature installation could no longer prepare that image correctly
- DISM and SFC do not necessarily validate every generated Sandbox VHD or cache file.

## Successful Sandbox Removal and Reinstallation

- The repair that finally worked was:
  - disable the Windows Sandbox optional feature
  - restart Windows
  - reset the generated Windows container data
  - enable Windows Sandbox again
  - restart Windows again
- This removed the broken Sandbox installation state and allowed Windows to regenerate it.

### Step 1: Disable Windows Sandbox

- Run in elevated PowerShell:

```powershell
Disable-WindowsOptionalFeature `
  -Online `
  -FeatureName Containers-DisposableClientVM `
  -NoRestart
```

- Restart Windows:

```powershell
Restart-Computer
```

### Step 2: Reset Generated Container Data

- After the restart, open elevated PowerShell.
- Stop the related services:

```powershell
Stop-Service vmcompute -Force -ErrorAction SilentlyContinue
Stop-Service hns -Force -ErrorAction SilentlyContinue
```

- The safer option is to rename the generated data:

```powershell
Rename-Item `
  "C:\ProgramData\Microsoft\Windows\Containers" `
  "Containers.sandbox-broken-20260615"
```

- Renaming is useful because the directory may also contain state used by:
  - Windows containers
  - Docker Windows-container workloads
  - other Windows container features
- If none of that data is needed, the directory can instead be deleted after Sandbox is disabled and the related services are stopped:

```powershell
Remove-Item `
  "C:\ProgramData\Microsoft\Windows\Containers" `
  -Recurse `
  -Force
```

### Step 3: Reinstall Windows Sandbox

- Enable the feature and its dependencies:

```powershell
Enable-WindowsOptionalFeature `
  -Online `
  -FeatureName Containers-DisposableClientVM `
  -All `
  -NoRestart
```

- Restart Windows again:

```powershell
Restart-Computer
```

### Step 4: Test a Plain Sandbox

- Launch Windows Sandbox without a custom `.wsb` file.
- Windows should regenerate:

```txt
C:\ProgramData\Microsoft\Windows\Containers
```

- The plain Sandbox launch succeeded after this flow.
- This was the confirmed fix for the `0x80070002` initialization failure.
- Keep the renamed backup until Sandbox and any Windows-container workloads have been tested.

## Recommended Final Order

- For an intermittent RDP disconnect with both AMD and NVIDIA GPUs:
  - test with the AMD iGPU disabled
  - restart after changing GPU state
  - run repeated Sandbox tests
- For initialization error `0x80070002`:
  - run DISM and SFC first
  - restart and retest
  - if the error remains, remove the Sandbox feature
  - reset the generated container data
  - reinstall Sandbox and restart
- When resetting `C:\ProgramData\Microsoft\Windows\Containers`:
  - stop Docker and other Windows-container workloads
  - rename the directory when rollback may be needed
  - delete it only when its other container data is not needed

- Avoid:
  - repeatedly changing unrelated RDP policy
  - manually copying system display DLLs without a verified package procedure
  - treating the unsupported `PreferredPhysicalGpu` value as a real Hyper-V setting
  - declaring success after a single launch
  - repeatedly toggling a display adapter without restarting Windows
  - deleting shared container data without considering Docker or Windows containers

## Final Conclusion

- The investigation contained two different failures that required different responses.
- For the original random disconnect:
  - the Sandbox VM started
  - RDP and graphics initialized
  - the session was terminated shortly after login
  - disabling the AMD iGPU stopped the observed disconnect
  - the best-supported diagnosis was a mixed-GPU display virtualization or driver-state problem
- For the later `0x80070002` failure:
  - Sandbox could not finish image preparation
  - DISM and SFC did not resolve it
  - removing and reinstalling the Sandbox optional feature, together with regenerating its container data, fixed it
- Repeated GPU disable and enable operations may have contributed to the later state failure, but this remains an inference.
- The confirmed final recovery for `0x80070002` was the full disable, restart, cache reset, re-enable, and restart sequence.
