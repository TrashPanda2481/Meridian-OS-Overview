# Engineering log

A reference of defects diagnosed and resolved during development. Each entry gives the symptom, how it was isolated, the root cause, the fix, and how the fix was verified. Items still awaiting final verification are marked. Smaller fixes are grouped in the table at the end.

---

## Display and boot

### i915 modesetting hang on stock Trixie kernels

- **Symptom:** display hangs during boot on Intel hardware; `i915` loads and the machine freezes before the desktop.
- **Diagnosis:** reproduced across every stock Trixie kernel available at the time (6.12.86 to 6.12.94); kernel log showed a symbol mismatch between `i915` and `drm_display_helper`. Booting with `nomodeset` reached the desktop but disabled kernel modesetting, confirming the fault was in the modesetting path.
- **Cause:** symbol mismatch in the stock kernel series, not in Meridian configuration.
- **Fix:** adopt the trixie-backports kernel (7.x line), pin it via APT preferences, remove `nomodeset`.
- **Result:** bare-metal validation on Dell Latitude 7490 (UHD 620) and Asus Zenbook (Iris Xe): modesetting active, `nomodeset` removed, desktop reached. The kernel is pinned to backports for this reason ([build-system.md](build-system.md)).

### Missing `cec.ko` on installed systems but not live

- **Symptom:** installed system hangs at boot; the live USB it was installed from boots normally.
- **Diagnosis:** the hang matched the earlier i915 dependency failure, pointing at kernel modules. `cec.ko` was present on the live system and absent on the installed one. The installer copies the live system with `rsync` plus exclude patterns; the difference was in the copy.
- **Cause:** the `media` exclude pattern had no leading slash, so it was unanchored and matched `/lib/modules/*/kernel/drivers/media/`, removing the media drivers (including `cec.ko`) during the copy.
- **Fix:** anchor exclude patterns with a leading slash.
- **Result:** clean installs boot with modesetting. Isolating this took three failed installs. The OTA asset mirror is written in pure Python with no `rsync` and no shell to exclude this class of path-matching data loss ([compass.md](compass.md#updates)).

---

## Installer

### FAT32 EFI label rejected

- **Symptom:** installer aborts at the partition step across builds b18 to b23.
- **Diagnosis:** `mkfs.fat` failed on the EFI partition. The label was `MERIDIAN_EFI` (and at one point `MERIDIAN EFI`), both 12 characters.
- **Cause:** a FAT32 volume label is limited to 11 characters.
- **Fix:** label the EFI partition `MERIDIANEFI` (11 characters).
- **Result:** installs complete through partitioning.

---

## Build and packaging

### Features dropped by `--apt-recommends false`

- **Symptom:** functionality missing from the image, a different set each build, with no build error. Affected firmware and Mesa drivers (b34), the audio stack (b36 to b41), and System Settings modules.
- **Diagnosis:** the build runs with `--apt-recommends false`. Each newly pulled-in package silently dropped anything it would have brought in through Recommends. A missing System Settings module presents as a Settings page that is blank or inert while the backing daemon still runs; `kcmshell6 --list` confirms the module is absent.
- **Cause:** Recommends-supplied packages were never installed.
- **Fix:** add each required package explicitly in `package-lists/base.list.chroot` with a comment. For System Settings gaps, add the specific `kde-config-*` package (resolved this way: the screen-configuration backend, the screen-locker module, the SDDM configuration module).
- **Result:** the affected subsystems (graphics, audio, Settings panels) function. The package list is maintained as an explicit record of packages that Recommends would otherwise supply ([build-system.md](build-system.md)).

### `firmware-linux` metapackage has no payload

- **Symptom:** a hook extracting `firmware-linux` produced no firmware blobs.
- **Cause:** in Trixie `firmware-linux` is a metapackage carrying only a changelog, not `.fw` files.
- **Fix:** use `firmware-linux-free` and `firmware-linux-nonfree`, which carry the blobs.
- **Result:** firmware present in the image.

### `libpam-kwallet` version (resolved by revert)

- **Symptom:** the KWallet PAM integration was changed from `libpam-kwallet5` to `libpam-kwallet6` on the basis that Trixie ships Plasma 6.
- **Cause:** Trixie's repositories do not carry the 6-series package.
- **Fix:** revert to `libpam-kwallet5`, which exists in the repositories.
- **Result:** the integration package resolves. Noted here because the version-matching reasoning was correct but the package did not exist; package availability in the target repository is checked before a version switch.

---

## Desktop, multi-monitor, and power

### Top panel placed on the wrong monitor (two identical displays)

- **Symptom:** on a two-monitor machine with identical displays, the top panel appears on the right-hand monitor at login instead of the primary.
- **Diagnosis:** the panel configuration and the screen model were both correct; the panel was placed and then moved. This is a view-placement race at session start, with no distinguishing attribute between two identical monitors to resolve the primary before placement.
- **Cause:** panel placement runs before the screen layout settles.
- **Fix:** a systemd user service adjusts the screen-priority setting after `plasmashell` settles. An earlier `xrandr` primary-output environment hack was ineffective and was removed.
- **Status:** fix in place and working in session; cold-reboot verification outstanding ([roadmap.md](roadmap.md)).

### Power menu and logout greeter inoperative

- **Symptom:** Sleep, Restart, Shutdown, and Logout do nothing; the logout confirmation screen does not appear.
- **Diagnosis:** two independent causes behind one symptom.
  1. The OTA asset mirror copied files into `/usr/share` preserving the source mode; some landed mode 640 (not world-readable), so session components could not read them.
  2. A build hook removed the Breeze Look-and-Feel to make room for Meridian's; the Breeze package was providing the logout-greeter QML, and no Meridian replacement existed.
- **Fix:** normalize synced shared files to `a+rX`; add the logout component to Meridian's own Look-and-Feel so it does not depend on Breeze.
- **Status:** both causes fixed. Outstanding: in a session predating the fix the greeter still renders no window; a full logout-login (or display-stack check) is needed to rule out an X11/Wayland display issue rather than packaging ([roadmap.md](roadmap.md)).

---

## Hardware

### NVIDIA Xid-109 crashes under load

- **Symptom:** the system runs, then crashes under sustained GPU load, logging `Xid 109`.
- **Diagnosis:** Debian's packaged 550 driver would not build against the kernel in use, moving the search to the upstream CUDA repository. The open-kernel-module 615 driver built but produced the Xid-109 crashes under load, so "builds" and "stable under load" diverged.
- **Cause:** the open-kernel-module path on this GPU generation is unstable under load; GSP firmware offload in that configuration is implicated.
- **Fix:** the closed-kernel-module 610.57.04 driver with GSP firmware disabled.
- **Result:** verified on an RTX 3060 under load. NVIDIA remains a post-install path, not an in-image one; the packaging (CUDA-repo sourcing, manual initramfs and modesetting glue) is unsuitable for the default image. A gated Compass-delivered rollout of the fix is designed and parked ([roadmap.md](roadmap.md)).

---

## Compass GUI and update plumbing

Smaller defects with cause and fix.

| Symptom | Cause | Fix |
|---|---|---|
| "Updates Available" panel always empty | the handler populated the widgets, then a state change rebuilt the page and destroyed them | set state first, populate second |
| Tray SIGSEGV after ~30 minutes | worker-thread references dropped mid-signal; `QThread(self)` left unparented at several sites | parent every worker thread; drop the stale done-slots |
| Streaming update workers leak child processes | a decode error left the subprocess unreaped | wrap every streaming worker in try/finally and reap the child |
| Install button used after the widget was freed | use-after-free in the finished handler | probe the button with try/except before touching it |
| Compass popup not dismissed on an outside click | an event filter only saw events inside the Compass process | use a `Qt.Popup`, which dismisses on any outside click |
| First-boot Skip button was a no-op | it did not advance the wizard | stop the worker, hide the progress bar, emit the completion signal |
| First-boot autostart silently misfired | the completion gate was in the `.desktop` `Exec=`, where the XDG-autostart generator drops `$HOME` | move the gate into the script |
| Cursor theme reverted to Breeze on first login | the Look-and-Feel default re-specified the Breeze cursor, overwriting the skel value | set the Look-and-Feel and SDDM cursor to the Meridian theme |
| Failed kernel upgrade silently un-held the kernel | the hold was not re-applied on failure when `--no-hold` was used | re-apply the hold on upgrade failure |
| Asset sync could change the mode of an arbitrary file | `os.chmod` follows symlinks on Linux | skip symlinks in the sync's permission pass |
| Wallpaper showed grey letterbox bars on smaller screens | skel pointed at the wallpaper directory; KDE picked the largest image | point skel at the correctly sized image with an edge-to-edge fill mode |

---

## Build bootstrap notes

From an earlier from-scratch (LFS) build, deferred in favour of the Debian base; retained as operational practice:

- Layout commands ran in a root shell where the build-target variable was unset, aiming at the host root instead of the target (no damage; operations were no-ops on an existing tree). Guard any root-level layout step with an explicit check that the target variable is set.
- A pasted heredoc with indented lines was not terminated correctly and hung the shell. Prefer line-by-line output over pasted heredocs for multi-line content.
