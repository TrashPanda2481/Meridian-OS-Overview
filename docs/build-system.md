# Build system

Meridian's ISO is produced with Debian live-build. The image is a Debian Stable base plus a chain of chroot hooks that lay down the theme, the daemon, the installer, the package debs, the icon pack, and the first-boot machinery. This document covers the pipeline, the hook discipline, the package-list rule that caused the most trouble, and reproducibility.

## Pipeline

The build runs on a Debian 13 VM with a single command:

```
cd build && sudo bash build.sh
```

live-build bootstraps a Debian chroot, installs the package lists, runs the numbered hooks in order, and packs the result into a bootable ISO with the Meridian GRUB theme, SDDM theme, Plymouth splash, and Compass preinstalled. The output is versioned `Meridian-OS-<iso>-b<n>.iso`; milestone builds get their own changelog entries and incremental builds are grouped.

## Hooks

Customisation lives in numbered chroot hooks rather than in edits to the base. The ordering matters, and getting it wrong has been a recurring source of bugs. Two examples that shaped the current discipline:

- **Order against user creation.** The cursor theme and the default Compass config were originally written only into `/etc/skel`. But the hook that creates the live `meridian` user with `useradd --create-home` runs before those hooks, so the live user never received either. The fix was to have those hooks also propagate into the already-created home, not just skel.
- **Autostart and a dropped environment.** The first-boot autostart gated itself on a `firstboot-complete` marker inside the `.desktop` `Exec=` line. The systemd XDG-autostart generator drops `$HOME` in that context, so the gate silently misfired. The gate moved out of the `.desktop` file and into the script itself.

Themed assets installed by the hooks are the same files that Stream 2's asset mirror later syncs to installed machines, so a single source of truth drives both the ISO and over-the-air updates. The mirror and its allowlist are in [compass.md](compass.md#updates).

## Package-list discipline

The build runs with `--apt-recommends false`, so apt installs no Recommends. On a desktop image a large amount of functionality arrives through Recommends, so each newly pulled-in package silently drops whatever it would have recommended. This produced no build error; it surfaced as missing features, a different set each time:

- Firmware, the Mesa GL drivers, KDE System Settings, and `pkexec` were dropped and had to be added back explicitly (build b34).
- The entire audio stack (ALSA UCM config, ALSA utils, PulseAudio utils, the Canberra plugins) went the same way on a later build and had to be restored.
- System Settings modules provided through Recommends kept vanishing, which presents as a Settings page that is blank or does nothing while the underlying daemon still runs. The diagnosis is `kcmshell6 --list`; the fix is to add the specific `kde-config-*` package to the base list.

The resulting rule: every omission that matters is an explicit addition in `package-lists/base.list.chroot`, with a comment saying why. The package list functions as an explicit record of packages that Recommends would otherwise supply.

A related case: `firmware-linux` in Trixie is a metapackage with no payload; a hook that extracted it got a changelog and no firmware. It was replaced with `firmware-linux-free` and `firmware-linux-nonfree`, which carry the `.fw` files.

## Reproducibility

Builds are pinned so the same commit produces the same image:

- Build-time APT is pinned to a Debian snapshot timestamp, so the package versions resolved during a build do not drift with the live mirror.
- The runtime APT sources are pinned to a stable mirror to eliminate CDN hash-mismatch failures that had been intermittently breaking builds.
- The kernel is pinned via an APT preferences file to the trixie-backports line (priority 600), with everything else at priority 1, so a kernel transition is a deliberate act and never an incidental one.

## "Compass owns updates"

The build masks the distribution's own update machinery so that nothing competes with Compass:

- `apt-daily.timer` and `apt-daily-upgrade.timer` are masked, so the system does not update itself on apt's schedule.
- CUPS was masked here until an audit found printing fully installed but unable to start because its units were masked; it was unmasked ([engineering-log.md](engineering-log.md)).

Every install also gets a swapfile created post-install, so a transient out-of-memory spike cannot take down the desktop session.

## A note on the from-scratch phase

Before the Debian-based build there was a throwaway Linux-From-Scratch phase, stopped at an early chapter in favour of the Debian base. Two operational practices are retained from it: verify the target environment variable in the exact shell that runs privileged layout commands (an unset variable aimed commands at the host instead of the build target), and prefer line-by-line echoes over pasted heredocs, which indentation can break. Both are in [engineering-log.md](engineering-log.md).
