# GitHub Issue Draft (EN) — Creality K3 · Silent Klippy Crash

**Repo target:** Creality official K-series Klipper/firmware repo covering K3 / CrealityOS3.
**Companion issue:** `GitHub_issue_K3_camera_USB_disconnect_EN.md` (USB camera port storm, same printer, same period).
**Delivery:** paste-ready. Full logs available on request (local archive: `k3-x2d-benchmark\证据\原始日志`).

---

## Title

K3 klippy process dies silently mid-print (no error / no traceback / no user notification) — occurred twice in 5 days across two firmware versions; interrupted job recorded as "complete" in print_stats

## Environment

- Printer: Creality K3, stock, purchased 2026-10 (JD.com self-operated), ~2 weeks in service
- Firmware: **4.0.2.88.13** (incident 1) → OTA to **4.0.3.1.5** on 2026-10-16 23:47 (image built 2026-10-11; confirmed via swupdate.log + /mnt/SDCARD/OTA) → **incident 2 occurred on the new firmware**
- Jobs: (1) PETG, 4h08m sliced duration, died at layer 54; (2) PETG, 2h32m sliced duration, died at ~2h
- Host access: read-only SSH (root)

## Summary

1. Twice within 5 days, the klippy process died silently in the middle of a print: no error, no traceback, no shutdown message, no app notification or push. Both deaths are the same signature.
2. The two deaths span an OTA upgrade — this is not a single-version regression.
3. After the two deaths, recovery behavior was inconsistent (watchdog restart without resume vs. full reboot with working resume). The user cannot predict whether a crash means "job lost" or "resume offered".
4. After death #2, `print_stats` reported the interrupted job as **"complete"** — the machine's own success accounting is wrong for crash-terminated jobs.

## Incident 1 — 2026-10-16 (firmware 4.0.2.88.13)

Timeline reconstructed from klippy.log / moonraker.log / 27_AppMonitor.log / 25_AppMultiMedia.log (all original files preserved):

| Time | Event | Source |
|---|---|---|
| 22:11:33 | Full system reboot (AppMonitor fresh start, camera re-detected 720p) | 27_AppMonitor.log, 25_AppMultiMedia.log |
| 22:13:03 | Klippy starts, print job begins | klippy.log `Starting Klippy` |
| 22:49:19 / 22:49:54 | Power-loss resume info written at layer 53 / 54 | klippy.log `record_power_loss_info` |
| **22:50:01.990** | **Klippy connection removed — process dead, silently** (no error, no shutdown, no traceback) | moonraker.log |
| 22:50:05 | Klippy restart #2 | klippy.log |
| 22:50:23 | **Connection removed again — second silent death, 18 s after restart** | moonraker.log |
| 22:50:33 | Klippy restart #3 — stable afterwards | klippy.log |

Pre-death Stats (last lines, healthy): sysload 2.06, memavail 330 MB, print_stall=0, all heaters on target.

Note: after these deaths there was **no full reboot and no resume entry point** — the job was simply lost. (User observed the stoppage in person; the nozzle oozed a small blob at the stop moment.)

## Incident 2 — 2026-10-18 (firmware 4.0.3.1.5)

| Time | Event | Source |
|---|---|---|
| 02:17:18 | Last normal Stats (roller print, ~2 h into a 2 h 32 m job) | klippy.log |
| 02:17:18–49 | Process dies silently — 25 trailing blank buffer lines, same recipe as incident 1 | klippy.log |
| 02:17:49 | solusp poll sees klippy `shutdown`; **print_stats shows "complete" although the job died at ~2 h / 2.5 h** | solusp.log |
| 02:17:55 | **Full system reboot** (uptime corroborates: 6 h 53 m at 09:11) | uptime |
| 09:08–09:13 | After boot, job auto-restarted and **resume worked correctly** from Z≈71 mm (layer ~297) | klippy.log |

Log growth before death: ~20 MB per 12 h (including rotated logs).

## What has been ruled out

- **Power supply / mains instability — refuted.** After incident 1 the printer was moved to a dedicated air-conditioner circuit; incident 2 recurred on the new circuit.
- **Simple OOM — unlikely.** memavail was 330 MB minutes before death #1; host did not appear memory-pressured. (K1 Max community reports of log-related instability make log growth worth checking — see Request #5.)
- **Single-version regression — refuted.** Deaths occurred on 4.0.2.88.13 and 4.0.3.1.5 respectively; the new firmware survived 21 h of continuous printing before the crash recurred.
- **Corrupted g-code file — refuted / separate.** In a separate event, the transfer checksum rejected a damaged file before printing (good). But see "Related state-machine issues" below.

## Open correlation (honest status)

The user observed 3 camera USB disconnects in the 40 s before death #1, suggesting a possible link between the camera USB storm (companion issue) and klippy death. That window's dmesg was lost to the subsequent reboots and **cannot be verified** — logged here as a lead, not a claim. Note that during the worst storm period (100+ re-enumeration cycles over 2 days), printing continued 20+ h with klippy fully stable, so the storm alone does not obviously kill the print service.

## Related state-machine issues (secondary, for triage)

1. **Zombie progress bar:** when a file was rejected as corrupted, klippy went to `standby`, but the screen kept showing a frozen print progress bar — UI state did not follow control state.
2. **No resource reclamation after aborted job:** after the same rejection, bed (70 °C), nozzle (140 °C) and fans stayed on indefinitely (observed >15 min, would have run all night) — no timeout, no auto-cool.
3. **PrintStats integrity:** an interrupted job recorded as "complete" (incident 2) inflates success metrics on the vendor side.
4. **"Root access" is read-only by design:** the rootfs is a read-only squashfs; `/etc/udev/rules.d`, `/run`, and `/etc/shadow` are all unwritable, so the factory root password (`creality_5733`, universal across units) **cannot be changed by the owner**. This is a security finding independent of the crashes — please advise.

## Requests to engineering

1. **Crash forensics path:** how can a user collect a core dump / crash log for klippy after a silent death? We will gladly run any diagnostic build or command and return results.
2. **Watchdog policy:** what determines restart-vs-full-reboot after a klippy death, and why was no resume offered on 2026-10-16 while resume worked on 2026-10-18? Predictable recovery matters more to users than crash-free perfection.
3. **Crash notification:** a klippy death currently produces no UI dialog and no push. Please surface "print terminated abnormally (code/state)" in the app.
4. **Fix print_stats accounting:** a job terminated by process death must not be recorded as `complete`.
5. **Check log-rotation / disk-pressure handling:** ~20 MB/12 h growth on the eMMC/SD; K1 Max history suggests log management has bitten this platform before.
6. **Heater timeout:** after any abnormal job termination, add a cooldown/timeout instead of indefinite hold.

## Attachments (available on request / to be bundled)

- klippy.log.2026-10-16 (38 MB, rotated) — incident 1, two deaths
- klippy.log / moonraker.log (2026-10-17/18) — incident 2
- solusp.log excerpt (02:17:49 poll showing "complete" mislabel)
- 27_AppMonitor.log / 25_AppMultiMedia.log (2026-10-16 22:11 reboot evidence)
- dmesg USB excerpt + hourly monitoring JSONL (USB storm, companion issue)

---

*Draft v1.0 · 2026-10-18 · All timestamps from original logs, double-sourced where possible · Chinese-language full analysis with community references available in the companion Zhihu articles linked from the camera issue.*
