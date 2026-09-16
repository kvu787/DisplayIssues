# Thunderbolt display startup investigation

Investigated September 16, 2026. Times below are local Pacific Daylight Time.

## Assessment

The strongest explanation is a **dGPU-only startup or restart initialization problem** in the laptop's dual USB-C DisplayPort output path. The specific faulty component is not yet established: Lenovo firmware, NVIDIA video firmware/UEFI display initialization, USB-C controller and graphics routing, the Windows NVIDIA driver, or their interaction with cable/monitor negotiation remain possible.

The decisive observations are that both monitors work when the second USB-C-to-DisplayPort cable is connected after Windows starts in dGPU-only mode, and both attached monitors boot successfully in hybrid/dynamic mode. Together these isolate a dependence on graphics mode and startup timing. A fixed inability to operate two displays is unlikely. A cable or firmware timing problem is still possible even when the same hardware works after startup.

**Scope: keep dGPU-only mode.** The user cannot use hybrid mode because of separate issues. Its successful boot is diagnostic evidence, not a proposed workaround. No further hybrid-mode tests are needed to establish this comparison.

| Graphics mode   | Connections at startup                    | User-observed result                   |
| --------------- | ----------------------------------------- | -------------------------------------- |
| dGPU-only       | Both USB-C-to-DisplayPort cables attached | Black-screen startup/restart failure   |
| dGPU-only       | One USB-C DisplayPort plus one HDMI       | Boots successfully                     |
| dGPU-only       | Second USB-C DisplayPort added afterward  | Both monitors operate                  |
| Hybrid/dynamic  | Both USB-C-to-DisplayPort cables attached | Boots successfully; excluded as a fix  |

The latest failed restart has a gap in Windows logging. It cannot yet be classified conclusively as a firmware/POST hang, an early Windows startup hang, a shutdown hang, or a running system with no working display. A lit laptop backlight displaying black pixels does not distinguish those cases.

No driver, firmware, display setting, or power setting was changed during this investigation. No reboot was initiated.

## Confirmed observations

- User: with both external monitors connected through the two Thunderbolt 5 ports, startup can stall with the laptop backlight on and an entirely black image.
- User: moving one monitor to HDMI permits startup.
- User: both USB-C-to-DisplayPort monitors work if the second is connected after Windows starts.
- User: both attached monitors boot successfully in hybrid/dynamic graphics mode; the observed startup issue is in dGPU-only mode. Hybrid mode is unavailable as a solution because of other issues.
- User: Windows+Ctrl+Shift+B and `pnputil` display resets were used several times this morning. This is relevant when interpreting diagnostic reports and device restarts.
- Each monitor/cable operating alone in each port, and unplugging one cable during an existing failed boot, have not been confirmed.

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

## Latest failed restart

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

1. **At the next naturally occurring black-screen stall, unplug one USB-C display without pressing the power button.** Record whether a logo, spinning dots, sign-in screen, or an already-running desktop appears, and the exact time. This distinguishes several possible stages without another forced shutdown.
2. **Compare firmware setup with one cable and then both cables connected before power-on.** From a fully shut-down machine, keep the lid open and use F2 at startup. If setup is reliably visible with one cable but stalls with both, focus on Lenovo firmware/controller/display negotiation; Windows driver reinstalls would not be the first test. Failure to see setup alone is not conclusive without the one-cable control.
3. **If firmware setup works with both, compare cold startup and Windows Restart.** A restart-only failure points toward shutdown/reset sequencing; failure as Windows takes over favors the Windows graphics path. Record screen transitions and timestamps rather than applying additional resets during the first observation.
4. **Verify each existing monitor/cable alone in each port.** If the problem follows one cable or port, investigate that path. If all four single-cable combinations work and only simultaneous attachment fails, shared initialization becomes more likely.
5. **If the failure reaches Windows, try one controlled display or driver change at a time.** A temporary 60 Hz setting for all displays and disabling monitor Adaptive-Sync can test mode/negotiation sensitivity. If necessary, compare the model-specific Lenovo NVIDIA driver 596.11 against 596.49. These are diagnostic comparisons, not verified remedies. Avoid combining driver changes, graphics-mode changes, and firmware updates in one test.

If both-cable startup remains reproducibly broken with each individual cable/port working, provide Lenovo this report and the reproduction steps. Ask specifically about **dGPU-only dual DisplayPort Alt Mode startup/restart, with hybrid startup and dGPU-only hot-plug working**, on Legion 9 18IAX10 with BIOS RZCN50WW, RTX 5090 Laptop GPU, video firmware 98.03.5e.00.57, and USB4 router firmware 60.60. This is more precise than reporting a generic black screen.

## Evidence files

- `Evidence/Hardware.json`: selected hardware and device properties.
- `Evidence/RelevantEvents.json`: selected restart, graphics, and PCIe records with original event data.
- `Evidence/DisplayConfiguration.txt`: display configuration extracted from DirectX diagnostics.
- `Evidence/RelevantLenovoCatalog.json`: relevant public catalog entries and download locations.
- `Evidence/Raw/`: ignored local diagnostic captures and retrieved source material, retained for follow-up. It includes full System/Application selections, both DirectX reports, the catalog response, and firmware readmes.

The evidence was collected from the running working configuration and historical logs. A controlled reproduction is still needed to identify the root cause.
