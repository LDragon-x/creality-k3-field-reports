# GitHub Issue Draft (EN) — Creality K3

**Repo target:** Creality's official K-series Klipper/firmware repository (confirm exact repo URL before posting; use the one covering K3 / CrealityOS3).
**Post timing:** ~~after D2 results~~ **Update 2026-10-18: D1/D2 results are in — see "Controlled-test results" below. Ready to post together with the companion issue `GitHub_issue_K3_silent_klippy_crash_EN.md`.**

---

## Title

K3 chamber camera triggers USB port storm ("disabled by hub (EMI?), re-enabling" loop) during PETG print at 36°C chamber temp; camera does not recover without reboot

## Environment

- Printer: Creality K3 (stock, ~2 weeks old) <!-- TODO: fill purchase date & channel -->
- Firmware: 4.0.3.1.5 (from printer screen, Settings → Device)
- Filament / job: PETG, 4h08m single-color print, bed 70°C, nozzle 235°C
- Chamber temp during issue: 36.2°C (no enclosure lid)
- Host access: read-only SSH (root)

## Description

During a long PETG print, the chamber camera on `usb1-port1` repeatedly disconnects and re-enumerates in a ~16-second cycle. Once started, the loop continues for hours and the camera stream does not recover until a full reboot. The printer itself is unaffected — klippy and moonraker stay alive and the print completes normally — but the app shows the camera offline / "device alarm" notifications repeat.

## Steps to Reproduce

1. Start a long PETG print (bed 70°C) with chamber temp above ~35°C.
2. Observe `dmesg` on the host.

## Expected Behavior

Camera stays connected for the duration of the print; if it does disconnect, it should recover without requiring a reboot.

## Actual Behavior

Kernel log (full excerpt attached, `dmesg_usb_excerpt_20261017.txt`):

```
[ 5136.937111] usb usb1-port1: disabled by hub (EMI?), re-enabling...
[ 5136.937136] usb 1-1: USB disconnect, device number 44
[ 5137.288163] usb 1-1: new high-speed USB device number 45 using sunxi-ehci
[ 5153.918335] usb 1-1: USB disconnect, device number 45
[ 5157.078320] usb 1-1: new high-speed USB device number 46 using sunxi-ehci
   ... (cycle repeats every ~16s, device number increments each loop)
[ 6741.879409] usb 1-1: new high-speed USB device number 58 using sunxi-ehci
```

- `lsusb -t` confirms the device on usb1-port1 is the stock chamber camera (`CREALITY CAM`, uvcvideo).
- `USB disconnect` count reached 15+ within the first storm hour; loop continued for hours.
- Print-side klippy Stats during the storm (excerpt from klippy.log):

```
Stats ... extruder: target=235.0 temp=235.1 ... heater_bed: target=70.0 temp=70.3 ... chamber_temp: temp=36.2 ... print_time=4713...
```

## Hypotheses Tested

1. **Power supply / power strip instability — refuted.** The failure first appeared on a shared power strip, so the printer was moved to a dedicated air-conditioner circuit and rebooted. The storm reappeared on the new circuit. Uptime confirms no reboot since the move.
2. **Chamber over-temperature — consistent, competing with #3.** Creality's own troubleshooting documentation for its camera system (K1 Max wiki) lists chamber temperature above 30°C as a known cause of camera disconnects. Chamber was 36.2°C when the storm started.
3. **Streaming-load correlation — new, open.** The storm self-stopped after ~1 h while printing continued (device number froze after 100+ cycles over 2 days). Both observed outbreaks coincided with active live-stream viewing; the camera stayed stable during long unattended prints. Thermal vs. streaming load could not be separated with available tooling — handed to engineering as the key experiment (continuous stream vs. no stream, same print).
4. **Software isolation — refuted (diagnostic value).** `usb unbind` only works momentarily: each hub port re-enable re-enumerates the physical camera as a new device and auto-rebinds the driver (device number kept climbing +22 cycles after unbind). There is no software-level defense while the device is attached.

## Controlled-test results (2026-10-17/18)

- **Scale:** 100+ disconnect→re-enumerate cycles over 2 days (device number 2→127+; allocator wraps, so true count is higher).
- **Print isolation:** during the worst storm, printing ran 20+ h continuously — klippy/moonraker stable, print unaffected. The damage is confined to the camera/app layer (stream dead, repeated "device alarm" notifications).
- **No self-recovery:** once looping, the camera never recovers without a reboot.

## Related Community Reports

- K1 Max "Camera Stops Working" long-running multi-user thread (camera + AI lidar appear to share a USB path): https://forum.creality.com/t/camera-stops-working/8148
- Hardware analysis: camera module SoC has no heatsink; thermal throttling → USB controller reset → no graceful recovery in firmware: https://github.com/Guilouz/Creality-Helper-Script-Wiki/discussions/531
- Same symptom reported by another K3 owner ("prints great but the camera disconnects sometimes"): https://www.facebook.com/groups/855804910568288/posts/887262047422574/

## Request

1. Can engineering confirm the root cause (thermal vs. cabling vs. USB power delivery)?
2. Is a firmware-side fix planned for graceful UVC recovery (the camera does not come back without a reboot)?
3. Given that the K3's official enclosure accessory targets ABS/ASA/PC — which require chamber temperatures far above the 30°C camera limit documented for this camera system — what is the supported configuration for enclosed high-temp printing with camera/AI detection active?

## Attachments

- `dmesg_usb_excerpt_20261017.txt` — raw kernel log (collection method noted inside)
- Hourly monitoring JSONL (usb device number / emi counters / chamber temp) — available on request
- klippy.log Stats excerpt during storm (camera dies, print healthy) — available on request

---

*Draft v1.1 · 2026-10-18 · v1.0 → v1.1: filled in controlled-test results (hypothesis #3 streaming correlation, software-unbind refutation, storm scale/self-recovery), updated post timing, cross-referenced companion crash issue.*
