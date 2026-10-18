# Creality K3 — Independent Field Reports

Independent, evidence-first field reports on a stock Creality K3 (purchased 2026-10, JD.com self-operated). Every timestamp is taken from original device logs (klippy.log / moonraker.log / solusp.log / dmesg / AppMonitor), double-sourced where possible. Raw logs and screenshots are archived locally and can be bundled on request.

This repository is not affiliated with Creality. It exists because two hardware/software defects reproduced on a two-week-old unit were met with scripted support responses instead of engineering answers. When Creality publishes the official K3 firmware repository (promised "in phases", not yet available as of 2026-10-18), these reports will be cross-posted there.

## Reports

| # | Date | Report | Summary |
|---|---|---|---|
| 1 | 2026-10-16/18 | [Silent klippy crash](2026-10-16_silent-klippy-crash.md) | Klippy process dies mid-print with no error, no traceback, no notification — three times in 3 days, across two firmware versions (4.0.2.88.13 → 4.0.3.1.5). Interrupted job recorded as "complete". Also documents a chronic MCU serial-corruption defect (36,525 protocol errors/day, live in standby). |
| 2 | 2026-10-17 | [Chamber camera USB storm](2026-10-17_camera-usb-storm.md) | Stock chamber camera enters a disconnect→re-enumerate loop (~16 s cycle, 100+ cycles over 2 days) once chamber temp passes ~36 °C — far above the 30 °C limit Creality documents for this camera system. Print unaffected; camera never recovers without reboot. |

## Key facts at a glance

- Both defects reproduced on a stock unit, read-only SSH access only. No modifications were ever made to the system.
- The USB camera storm and the klippy crash are tracked as two independent defects; a possible correlation on 2026-10-16 is logged as an unverified lead (dmesg window lost to reboot).
- Recovery after klippy death is inconsistent: 2026-10-16 → watchdog restart, no resume offered; 2026-10-18 → full reboot, resume from Z≈71 mm worked. Same crash signature.
- Firmware OTA (10-16 23:47, 4.0.2.88.13 → 4.0.3.1.5, image built 2026-10-11) separates the two crashes across versions — not a single-version regression.

## Full analysis (Chinese)

- [声明：本系列的评测准则、行为边界与升级政策](https://zhuanlan.zhihu.com/p/2094860293990106967)
- [番外篇：一台 K3、两种病灶——「无报错停机」的完整推理链](https://zhuanlan.zhihu.com/p/2094860735100858616)

## Status

- 2026-10-18: Reports 1–2 published. A formal technical letter with a 7-working-day response deadline was sent to Creality customer service on 2026-10-17.
- Attachments (raw logs, screenshots, monitoring data) available on request via issue comments.
