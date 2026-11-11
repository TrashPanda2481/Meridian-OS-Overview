# Roadmap

What is deferred, what is open, and what comes next. Dates are not committed.

## Open and in progress

- **The Compass Rust port.** The Python implementation is the working prototype; Rust is the production target. The layering is already shaped for it (output goes through one layer, the daemon's hot path has no disk I/O, the port runs from the no-privilege crate upward). A security driver: the port removes the Python interpreter from the privileged path. Scoped, not started.
- **The Navigate-bearing safety gate.** Full automation currently emits its actions without a one-time confirmation that automation is intended. The kernel update stream already shows the pattern (confirm even on the fully automated bearing); the general gate, a per-session first-use confirmation recorded to config plus an automation log, is designed and not yet built.
- **OTA repository permission hardening.** Tightening the file permissions on the root-owned update repository is in progress. It is described at the level of the area rather than the method while it is open; see [security-model.md](security-model.md).
- **Two fixes awaiting final confirmation.** The multi-monitor panel-placement service works in session but is owed a clean cold-reboot check. The power-menu and logout-greeter fix has both root causes addressed but is owed a full logout-login to rule out a display-stack issue in pre-fix sessions. Both are in [engineering-log.md](engineering-log.md).
- **Installer: set the root password.** The root account ships locked; the installer is the right place to prompt for and set a root password and does not yet.

## Deferred by choice

- **NVIDIA as an in-image driver.** The stable configuration is known and verified (the closed 610.57.04 driver with GSP firmware disabled, on the hardware tested), but the packaging is a minefield that does not belong in a default beginner image. It stays a post-install path. A fleet rollout of the fix as a self-gating Compass update is designed and parked pending an update-UX rework.
- **Installer scope.** Network and Wi-Fi configuration during install (handled post-install by the Plasma network manager), disk encryption, and dual-boot alongside another OS are all deferred.
- **The proprietary driver path in general** is a post-install Compass feature rather than an image default, to keep the default image reliable for the beginner end of the audience.

## Planned desktop and integration work

The current GUI is self-contained but built with a floating toolkit window; several integrations would make it feel native to Plasma:

- A proper Plasma applet to replace the floating popup, so it can dock in the panel.
- Native KDE notifications in place of the custom toast.
- A System Settings module for the bearing selector.
- A desktop-wide `compass explain` through KRunner.
- A full Look-and-Feel package so the Meridian identity is a first-class Plasma theme rather than a set of overlaid assets.

A Meridian-branded System Settings hub (curated deep links to the relevant KDE modules plus native Meridian panels, rather than a reimplementation of System Settings) is an active sub-project on its own branch.

## Related projects

Projects that share components or design with Meridian:

- **Grid**, a terminal written in Rust, intended as the first external consumer of the Compass crates once the port produces them.
- **The Meridian Look-and-Feel** (icon pack and theme), packaged as Debian packages and delivered both in the image and over the air.

Tracked in their own repositories.
