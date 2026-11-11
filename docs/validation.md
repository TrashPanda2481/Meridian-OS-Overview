# Validation

Meridian is validated on real hardware, not only in a virtual machine. "Validated" here means a specific, checkable list was confirmed on the machine, not that it booted once. The install path is re-confirmed on bare metal at each milestone.

## Hardware

| Machine | CPU | GPU | Result |
|---|---|---|---|
| Dell Latitude 7490 | Core i5, Kaby Lake Refresh | Intel UHD 620 (Gen 9.5) | Full validation |
| Asus Zenbook | Core i7-1165G7, Tiger Lake | Intel Iris Xe (Gen 12) | Full validation; a cosmetic Plymouth offset on this GPU |
| Desktop (Gigabyte B760M) | Core i5-14400, Raptor Lake | NVIDIA RTX 3060, proprietary driver | Primary development machine, in daily use; dual monitors; NVIDIA via the post-install Xid-109 fix |

The two laptops use Intel graphics, the primary target for the default image. The desktop runs a discrete NVIDIA GPU on the proprietary driver applied post-install (the closed 610.57.04 driver with GSP firmware disabled), the configuration from the Xid-109 investigation in [engineering-log.md](engineering-log.md). NVIDIA remains a post-install path, not part of the default image.

## What a full validation covers

Confirmed on the hardware above:

- **Boot chain.** A DD-mode USB writes a bootable stick; Plymouth splash; the themed GRUB menu; the themed SDDM greeter leading to the Plasma desktop with the wallpaper and no grey letterbox bars.
- **Graphics.** `i915` kernel modesetting active, with `nomodeset` removed, on the pinned backports kernel.
- **Compass.** compassd `active (running)`; the bearing reads correctly; the tray icon shows the bearing.
- **Power and devices.** Battery and power indicator in the tray; CPU governor set to performance at boot; audio working.
- **Packages and updates.** Package search and install from the Compass GUI; the `--system`, `--meridian`, and `--kernel` update streams, including the kernel stream's hold detection.

## Install path

The installer is re-validated on bare metal at each milestone, end to end: partition, copy, user and password creation, EFI bootloader, then first boot through the wizard (bearing, network, update, finish). The shell hook's inline guidance over D-Bus is confirmed with no job-control noise, and the command-not-found suggestion is confirmed to fire only on the right exit code.

The current release, 1.1-b5, is the first publicly shipped build of the 1.1 line and closed a set of gaps that only appeared once the system was installed rather than run live: live-session autologin cleanup so the greeter does not fall through, restored time sync and crash capture, persistent journald, a durable hibernate opt-out guarded against an undersized swapfile, and correct propagation of the skeleton configuration (autostart and wallpaper) to the installed user.

## Known issues that do not stop daily use

These are logged and are cosmetic or bounded.

- **A couple of kernel log lines at boot.** `drm_display_helper` logs CEC symbol errors because CEC is built into the backports kernel rather than shipped as a loadable module, so there is no `cec.ko` for the extra-modules package to provide; `i915` loads cleanly regardless. Separately, an Intel LPSS device logs a probe error (`-16`). Both are cosmetic and unrelated to the display coming up.
- **The root account ships locked.** `su` fails until a root password is set by hand (`sudo passwd root`). The installer is the right place to handle this and does not yet.
- **The Plymouth offset on Tiger Lake** is a cosmetic splash-positioning issue, not a boot failure.

## Verification

Fixes are verified against the specific failure they address, and the verification is recorded with each defect in [engineering-log.md](engineering-log.md). Items still awaiting confirmation (a cold-reboot check on the multi-monitor panel fix, a full logout-login check on the power-menu fix) are marked as such.
