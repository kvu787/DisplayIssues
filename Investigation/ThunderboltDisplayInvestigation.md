# Thunderbolt display startup investigation

Investigated September 16, 2026. Times below are local Pacific Daylight Time.

## Assessment

The strongest explanation is a **dGPU-only startup display initialization problem**, with the latest experiments increasing suspicion of the firmware-controlled display path. The user now distinguishes two behaviors: a TB5 DisplayPort monitor already attached at startup prevents a visible BIOS setup screen, whereas two attached TB5 monitors cause an apparent Windows boot failure. In contrast, attaching one display after entering visibly working BIOS setup leaves the internal setup screen working and produces no external picture. These symptoms may share a cause, but that is not established. Lenovo UEFI, NVIDIA video firmware/Graphics Output Protocol (GOP), USB-C display routing, and cable/monitor negotiation are the leading areas to investigate. The Windows NVIDIA driver remains relevant to the later boot/handoff symptom, but would not itself draw the BIOS setup screen.

The decisive observations are that both monitors work when the second USB-C-to-DisplayPort cable is connected after Windows starts in dGPU-only mode, and both attached monitors boot successfully in hybrid/dynamic mode. Together these isolate a dependence on graphics mode and startup timing. A fixed inability to operate two displays is unlikely. A cable or firmware timing problem is still possible even when the same hardware works after startup. In a subsequent test, disconnecting the displays during an existing dual-TB5 failure did not recover output: the laptop remained lit with a black image. This demonstrates a persistent failure for that attempt, but does not alone distinguish a system boot hang from a graphics subsystem that cannot recover output.

**Scope: keep dGPU-only mode.** The user cannot use hybrid mode because of separate issues. Its successful boot is diagnostic evidence, not a proposed workaround. No further hybrid-mode tests are needed to establish this comparison.

| Graphics mode   | Connections at startup                    | User-observed result                   |
| --------------- | ----------------------------------------- | -------------------------------------- |
| dGPU-only       | Both USB-C-to-DisplayPort cables attached | Black-screen startup/restart failure   |
| dGPU-only       | One USB-C DisplayPort plus one HDMI       | Boots successfully                     |
| dGPU-only       | Second USB-C DisplayPort added afterward  | Both monitors operate                  |
| Hybrid/dynamic  | Both USB-C-to-DisplayPort cables attached | Boots successfully; excluded as a fix  |

The initially investigated 09:48 restart has a gap in Windows logging. It cannot yet be classified conclusively as a firmware/POST hang, an early Windows startup hang, a shutdown hang, or a running system with no working display. A lit laptop backlight displaying black pixels does not distinguish those cases. Later experiments require their own timestamps to be associated with specific Windows records.

The agent has not changed drivers, firmware, display settings, or boot settings, and has not initiated reboots. The user has performed connection and boot experiments separately. Guidance about an optional boot-status-policy override was provided; its application has not been verified.

## Confirmed observations

- User: with both external monitors connected through the two Thunderbolt 5 ports, startup can stall with the laptop backlight on and an entirely black image.
- User: moving one monitor to HDMI permits startup.
- User: both USB-C-to-DisplayPort monitors work if the second is connected after Windows starts.
- User: both attached monitors boot successfully in hybrid/dynamic graphics mode; the observed startup issue is in dGPU-only mode. Hybrid mode is unavailable as a solution because of other issues.
- User: Windows+Ctrl+Shift+B and `pnputil` display resets were used several times this morning. This is relevant when interpreting diagnostic reports and device restarts.
- User follow-up: connecting displays to both TB5 ports causes apparent boot failure; every other tested display configuration boots. The exact list of individual cable/port permutations has not been recorded.
- User follow-up: with a display attached to a TB5 port before startup, BIOS setup produces no visible image on either the laptop or external monitors. The user believes setup is still running because fans become audible and approximately one second of power-button hold switches the machine off, compared with approximately five seconds in the other state.
- Those fan and power-button observations support the user's interpretation on this machine, but are indirect indicators rather than proof that setup is fully responsive. Display loss and a machine hang remain distinguishable possibilities.
- User follow-up: disconnecting the displays without powering off during the dual-TB5 failure left the laptop screen lit and all black; no desktop or visible boot progress appeared. Exact attempt time and wait duration were not supplied.
- User follow-up: with all external displays disconnected, BIOS setup was visibly working on the laptop. Connecting one display to a TB5 port while remaining in setup produced no picture on the external display, left the BIOS picture on the laptop unchanged, and left setup apparently functioning normally. The specific port/cable used and a second-display-in-setup test were not supplied.
- Disconnecting displays specifically while in the presumed invisible BIOS setup state remains unconfirmed. The failed unplug test above concerns the normal boot attempt, not that separate setup-state experiment. The visible-setup hot-plug test is now completed and should not be proposed again as untried.

## Follow-up: invisible BIOS setup versus apparent boot failure

The user's later experiments change how the earlier F2 test should be interpreted. A blank screen after requesting setup cannot be treated as failure to enter setup. In particular, the one-TB5 arrangement reportedly boots Windows normally despite its missing BIOS picture.

| dGPU-only connection at power-on | BIOS setup picture, user report | Windows startup, user report |
| -------------------------------- | ------------------------------- | ---------------------------- |
| No TB5 display                   | Visible in tested arrangement   | Works                        |
| One TB5 display                  | No visible output on any screen | Works                        |
| Two TB5 displays                 | No visible output on any screen | Apparent boot failure        |

The GPU's firmware display implementation runs before the Windows native display driver takes over. Microsoft describes that transition in [Boot Screen Components](https://learn.microsoft.com/en-us/windows-hardware/drivers/bringup/boot-screen-components) and [Providing Seamless State Transitions](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/seamless-state-transitions-in-wddm-1-2-and-later). These explain the diagnostic boundary; they do not identify a known defect in this particular Lenovo model.

The leading hypothesis is firmware display initialization or output selection that behaves differently in dGPU-only mode. A dual-output interaction may additionally stall boot or leave Windows without a visible display. Because display output is already unreliable before Windows, blankness alone is now especially weak evidence that the operating system failed to start. A completely separate Windows handoff issue also remains possible.

The subsequent visible-setup hot-plug test adds an attachment-order distinction. Starting with no external displays establishes a working internal firmware display; connecting one TB5 display afterward does not disrupt it. Having a TB5 display present during startup produces the earlier invisible-setup behavior. This is consistent with a problem in how firmware discovers or configures attached displays during startup. It does not establish the exact implementation or prove that the external display was detected or activated during the hot-plug test: it never showed an image. In particular, the result neither demonstrates a generic hot-plug failure nor clears the firmware/controller path of a startup-only problem.

The new test confirms that BIOS can remain responsive with a TB5 cable connected after successful initialization. It does not prove that the earlier black-screen setup attempts were responsive; that separate inference still rests on the user's fan and power-button observations.

A read-only follow-up collection at approximately 11:07 PDT found additional boot and unclean-shutdown records from the experiments. Those records do not identify the attached cables or whether F2 setup was requested, so they cannot assign a failure to a particular configuration without matching experiment timestamps. They are retained locally in `Evidence/Raw/FollowUpBootExperiments.json`.

At approximately 11:10 PDT the user reported that disconnecting the displays during the failed normal boot did not restore output. A further read-only log collection found the latest recorded Windows kernel start at 11:00:35 and a logon notification at 11:01:10. No exact timestamp was supplied for the failed unplug experiment, so those events cannot be assigned to that failed attempt or used to establish whether Windows was running during it. The collection is retained in `Evidence/Raw/DisconnectTestBootEvents.json`.

The failed unplug test makes a simple recoverable output-selection problem less likely. It does not rule out Windows running with a stuck graphics subsystem. Independent evidence of operating-system activity, or a precisely matched failed-attempt timeline, is needed for that distinction. There is no reason to keep repeating the same unplug test without changing another meaningful condition.

These cables use DisplayPort Alt Mode on USB-C; a Thunderbolt-capable socket does not make the cable a Thunderbolt display tunnel. Microsoft distinguishes these display connection types in its [USB4 requirements](https://learn.microsoft.com/en-us/windows-hardware/design/component-guidelines/usb4-design-details-and-general-requirements).

## Machine and working configuration

| Component              | Observed value                                                     |
| ---------------------- | ------------------------------------------------------------------ |
| Computer               | Lenovo Legion 9 18IAX10, machine type 83EY                         |
| Operating system       | Windows 11 Pro 25H2, build 26200.9457                              |
| BIOS                   | RZCN50WW; Lenovo README dated June 25, 2026                        |
| Active graphics        | NVIDIA GeForce RTX 5090 Laptop GPU                                 |
| NVIDIA driver          | 596.49 / Windows version 32.0.15.9649                              |
| NVIDIA video firmware  | 98.03.5e.00.57                                                     |
| USB4 host              | Intel PCI device 8086:5781; Microsoft USB4 host/router drivers     |
| USB4 driver            | 10.0.26100.9278                                                    |
| USB4 router firmware   | Device property reports 60.60                                      |
| USB-C policy interface | UCM-UCSI ACPI Device, no current problem code                      |
| Laptop display         | BOE NE18NZ2, 3840 × 2400 at 240 Hz                                 |
| External displays      | Two ASUS PA278QGV, both currently 2560 × 1440 at 120 Hz            |
| Current connections    | Embedded DisplayPort, external DisplayPort, HDMI; extended desktop |
| Current color mode     | SDR reported for all three displays; HDR capability present        |
| Windows Fast Startup   | Disabled: HiberbootEnabled = 0                                     |

DirectX diagnostics associate all three active displays with NVIDIA and report the mux target as dGPU. No active Intel display adapter was enumerated. This agrees with the user's confirmation of dGPU-only mode; BIOS configuration itself was not changed. Current healthy device status describes the working HDMI-plus-DisplayPort arrangement, not the failing startup arrangement.

Lenovo specifies up to four independent displays including the built-in panel. That supports treating this as abnormal behavior, although the specification alone does not prove every cable/topology combination. See [Lenovo's model specification, page 2](https://psref.lenovo.com/syspool/Sys/PDF/Legion/Legion_9_18IAX10/Legion_9_18IAX10_Spec.pdf). ASUS lists DisplayPort and HDMI inputs for the [PA278QGV](https://www.asus.com/displays-desktops/monitors/proart/proart-display-pa278qv-gen2-pa278qgv/techspec/).

## Initially investigated failed restart

| September 16 time | Evidence                                                                          |
| ----------------- | --------------------------------------------------------------------------------- |
| 09:48:11          | Windows Explorer requested a restart, System event 1074.                          |
| 09:48:32          | Lenovo Dispatcher did not finish its preshutdown handling, event 7043.            |
| 09:48:32          | Windows Event Log service stopped, event 6006.                                    |
| 09:49:51          | Power-button timestamp later recorded in Kernel-Power event 41.                   |
| 09:50:41          | Next recorded Windows kernel startup.                                             |
| 09:50:41          | Kernel-Boot event 20 reports previous shutdown and previous boot unsuccessful.    |
| 09:50:43          | Kernel-Power event 41: unclean shutdown, BugcheckCode = 0.                        |
| 09:50:52          | Windows logon notification.                                                       |
| 09:57:00          | Next recorded boot reports the preceding boot and shutdown successful.            |

No intervening kernel-start event was found between 09:48:32 and 09:50:41 in the collected System log. This supports failure around the restart/early-boot transition, but missing logging cannot prove exactly where execution stopped. Event 41 confirms an unclean interruption; it is not a diagnosis of defective power hardware. No blue-screen stop code is recorded for this interruption.

Lenovo Dispatcher also times out during successful restarts, including at 09:56:36, so that service event does not explain the monitor-dependent failure by itself.

## Graphics and hardware evidence

On September 15, NVIDIA logged event 153 at 15:54:12 and 15:58:08 with the event payload `Resetting TDR occurred on GPUID:200`. Windows Error Reporting also references `0x117` live dumps named `WATCHDOG-20260915-1554.dmp` and `WATCHDOG-20260915-1558.dmp`. Microsoft defines [0x117](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x117---video-tdr-timeout-detected) as a display-driver response timeout. Those are useful graphics-failure clues, but the exact monitor configuration and any deliberate reset activity at those times are unknown.

Reports referencing 09:39, 09:44 and 09:45 today include `0x1A8` and `0x1B8` black-screen live dumps. A DWM report explicitly identifies `WindowsBlackScreenDiagnosticsV1` with trigger `Hotkey`. Given the user's confirmation of manual reset attempts, these must not be counted as independent spontaneous crashes. Microsoft's [0x1A8 reference](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x1a8--video-dxgkrnl-black-screen-livedump) and [0x1B8 reference](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x1b8--video-miniport-black-screen-livedump) describe diagnostic live dumps, including hotkey triggers. Report submission times can be much later than the dump filename's capture time.

One corrected PCIe error occurred September 15 at 21:39:13 on root port `8086:7ECA`, bus 0/device 6/function 1. Its location path is also the upstream prefix of the USB4 host's path. This is a relevant secondary clue, not proof of a bad Thunderbolt controller or the cause of this morning's failure. Another 25 corrected PCIe events involve a different device, `8086:272B`; they should not be lumped into the Thunderbolt count.

Recurring USB host companion warnings, Intel platform-driver startup warnings, BitLocker messages, and historical driver-update failures were also present. They lack a unique correlation to the failing connection arrangement. The Intel platform participant and USB4/UCSI devices currently report no problem codes. Ordinary DCOM warnings were excluded from the evidence selection.

Windows denied access to the protected live-kernel dump directory under the available process privileges. Consequently, no kernel dump was opened or stack-analyzed; the dump references above come from readable event logs. The current evidence does not establish a particular failing driver instruction or hardware defect.

## Firmware and driver comparison

The live [Lenovo catalog for this machine](https://pcsupport.lenovo.com/us/en/products/laptops-and-netbooks/legion-series/legion-9-18iax10/83ey/downloads/driver-list) was retrieved on September 16:

- **BIOS DS576006:** RZCN50WW, matching the installed BIOS. Its published README contains no substantive change list. No newer system BIOS was listed, and no confirmed dual-display fix was documented there.
- **NVIDIA DS575995:** Lenovo offers 32.0.15.9611, equivalent to 596.11. The installed 596.49 is newer. Lenovo's driver is a possible controlled comparison, not a newly available upgrade or a proven fix.
- **Management Engine DS580490:** despite the catalog title “Flash BIOS update,” its README identifies a separate ME firmware package, RZME50WW / 19.0.11.2289. It documents security fixes, not this display issue. Installed ME firmware was not established; the installed MEI driver version is a different thing.
- **Lenovo Dispatcher:** installed 3.2.0.26 matches the catalog.
- **Intel platform framework:** installed IPF 2.3.20303.5058 matches the IPF component listed in Lenovo's DPTF package.
- No separately named Thunderbolt/USB4 firmware update appeared in this model's returned public catalog. This does not establish whether controller firmware is included in another package.

## Focused next tests

Keep dGPU-only mode throughout. Use the working HDMI-plus-DisplayPort configuration, or connect the second USB-C display after startup, as the practical workaround. The following tests require a convenient restart window and physical cable access; none was performed remotely.

1. **Visible-BIOS hot-plug test completed: internal setup remains visible and responsive; external display stays blank.** Do not repeat it as an untried diagnostic. The relevant remaining distinction is attachment before firmware initialization versus attachment afterward. The external output not activating prevents this test from validating the complete external-display path.
2. **If already in the apparently active but invisible setup state, disconnect all TB5 displays without restarting.** Seeing the setup screen immediately reappear would strongly confirm that setup was active with no usable output. No image appearing is inconclusive because firmware may not reinitialize the internal display after hot-unplug.
3. **Normal-boot unplug test completed: no visible recovery.** Do not repeat it as an untried diagnostic. If further normal-boot experiments are needed, record the attempted startup and forced-off times so the logs can distinguish a failed attempt from the later successful recovery boot. Evidence of Windows activity during the black screen would support a persistent graphics failure; absent log entries alone would not prove Windows never ran.
4. **Keep a short record of the individual cable/port tests and cold-start versus restart outcomes.** The user's broad comparison already identifies dual-TB5 attachment as the trigger; precise permutations help Lenovo reproduce it without repeating unnecessary tests.
5. **Prioritize firmware/display-output diagnosis for the BIOS symptom.** Windows refresh-rate changes or driver reinstalls do not directly control setup rendering. If evidence separately places the dual-display failure at the Windows graphics handoff, a temporary 60 Hz configuration, monitor Adaptive-Sync comparison, or model-specific Lenovo NVIDIA 596.11 driver comparison can be considered one at a time. These remain diagnostic comparisons, not verified remedies.

The current findings justify reporting a firmware/display-output issue to Lenovo. Include the attachment order explicitly: **in dGPU-only mode, a TB5 DisplayPort monitor present at startup makes BIOS setup invisible on all screens. Entering setup with no external displays and then attaching one TB5 display leaves the internal setup working but produces no external picture. Two TB5 monitors present at startup prevent a visible successful Windows boot; disconnecting them during that failed attempt does not restore output. Hybrid startup and attaching the second display after Windows starts work.** Identify Legion 9 18IAX10, BIOS RZCN50WW, RTX 5090 Laptop GPU, video firmware 98.03.5e.00.57, and USB4 router firmware 60.60. Describe BIOS activity during the invisible-setup attempts as inferred from repeatable fan/power-button behavior; the successful visible-setup test is a separate, directly observed condition.

## Evidence files

- `Evidence/Hardware.json`: selected hardware and device properties.
- `Evidence/RelevantEvents.json`: selected restart, graphics, and PCIe records with original event data.
- `Evidence/DisplayConfiguration.txt`: display configuration extracted from DirectX diagnostics.
- `Evidence/RelevantLenovoCatalog.json`: relevant public catalog entries and download locations.
- `Evidence/Raw/`: ignored local diagnostic captures and retrieved source material, retained for follow-up. It includes full System/Application selections, both DirectX reports, the catalog response, and firmware readmes.

The evidence was collected from the running working configuration and historical logs. A controlled reproduction is still needed to identify the root cause.
