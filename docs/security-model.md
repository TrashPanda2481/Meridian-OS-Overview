# Security model

Meridian's guidance layer exposes session-bus methods and produces advice the user is encouraged to act on, so its security is designed around that surface. This document covers the privilege model, the hardening that shipped, and the audit of the inter-process surface. It describes fixed issues and design constraints; it does not publish working exploit steps for anything still open.

## Design rule: guidance is not permission

This design rule is also a security property. A user's guidance level (the bearing) never grants or revokes administrative authority; only the amount of explanation and confirmation differs. No bearing unlocks a capability, so no bearing is worth attacking to unlock one.

## Privilege model

compassd holds no privilege.

- **compassd runs as the user, always.** Its systemd user unit carries a strict sandbox: `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, the `ProtectKernel*` set, `PrivateDevices`, `RestrictSUIDSGID`, and `LockPersonality`. It keeps an in-memory session buffer and a rule engine, and it never calls apt, dpkg, or any system tool.
- **Privileged work is delegated, not held.** When a privileged action is needed, it goes through polkit and pkexec to a small bash helper that accepts only an allowlisted set of modes. pkexec clears the environment, which closes the `PYTHONPATH`-injection path into the privileged process. The polkit actions require the administrator credential every time; there is no cached "you authenticated a minute ago, go ahead" window.

## Hardening that shipped

### The polkit authorisation gap

A hardening pass across the build tree found that the polkit actions for the privileged helpers (install a package, run a system update) were configured to require `auth_self_keep` rather than `auth_admin_keep`. The practical effect is serious: `auth_self` lets any locally active user authorise the action with their own password, rather than requiring an administrator's. On a multi-user machine that is a local privilege escalation. The actions were changed to `auth_admin_keep`. This was the most severe of a set of fixes in that pass, which also corrected an invalid D-Bus policy syntax and removed dead configuration.

### The inverted systemd hardening

An audit found the hardening effort had gone to the smallest blast radius and skipped the largest. compassd, an unprivileged user service, carried ten sandboxing directives. The two services that actually run as root, the CPU-governor setter and the VirtualBox-additions helper, carried none.

The fix was not a copy-paste of the user unit's directives, because the root services have real jobs that generic lockdown would break. The CPU-governor service exists precisely to write `scaling_governor`, so a blanket `ProtectKernelTunables=yes` would stop it working; it got the full lockdown (empty capability set, no network, private devices) plus a carve-out that returns only the cpufreq tree as writable. The VirtualBox-additions helper compiles kernel modules and is inherently harder to confine, so it got the conservative set that leaves its build and install paths untouched, and it only runs inside a VirtualBox guest anyway. Both root units are now hardened to their actual needs.

### The interpreter-as-root question, and the checker built for it

The privileged update path is `pkexec` to a bash helper to a Python process running as root. The bash helpers are sound (strict shell flags, allowlisted modes, cleared environment). The remaining exposure is on disk: any directory on root's Python import path, or any bytecode cache beneath it, that a non-root user can write is root code execution on the next run.

The project ships a checker: a script that, run as root on an install, flags any directory or importable file on root's import path that a non-root user owns or can write (ownership alone counts, since an owner can change the mode). This is a motivation for the planned Rust port, which removes the interpreter from the privileged path; the port is ordered by privilege for this reason.

## The inter-process surface, audited

compassd exposes methods on the session D-Bus and feeds their input into a suggestion engine. This surface was audited; the findings are summarised here, and the detailed audit is in the private tree.

The threat model, in plain terms: on a shared session bus, any process running as the same user can, absent a policy, call these methods. The risks that follow are not about crashing the daemon; they are about poisoning what it says.

| Concern | What it is | Status |
|---|---|---|
| Unauthenticated method access | without a policy file, any same-user process can call the methods | session policy file now ships; caller-identity verification is a requirement for the Rust implementation |
| Session-buffer poisoning | injected "commands" could trigger suggestions that reference harmful follow-ups as if the user had run something successfully | input length and encoding validation added; caller verification carried into the Rust design |
| Suggestion-rule hot-swap | the reload method re-reads rules with no integrity check | production requirement: rules installed to a root-owned system path and verified against a reference hash before loading |
| Regex denial-of-service | untrusted input into a backtracking regex engine could stall the event loop | input bounds added; the Rust implementation mandates a linear-time regex engine, eliminating the class |
| Unconstrained window-title input | a method meant for the window-manager observer accepts arbitrary strings from any caller | requirement: tag observer entries by source and exclude them from command-pattern matching |
| apt without a privilege boundary | a future automation path must never run package operations inside the daemon | requirement: a separate, allowlisted, polkit-gated helper; the daemon must never call apt directly |

Two of these were closed immediately because they were nearly free: the session policy file (zero code) and a hard exit with a Compass-formatted error when a package write is attempted without root (the previous behaviour only printed a warning and let apt fail with a raw error). The rest are recorded as hard requirements for the Rust implementation, so they are constraints on the production target.

## Fail-safe posture and limits

- **Updates are owned, not automatic in the background.** The distribution's own auto-update timers are masked, so nothing modifies the system outside Compass. The kernel is held by default and its upgrade confirms on every bearing, including the fully automated one, so an untested kernel cannot land on a working machine unattended.
- **Not tamper-proof against root; tamper-evident instead.** The model assumes an administrator can perform administrator actions. Privileged actions are authenticated every time and show the real command. The model defends against code running as the ordinary user gaining more than the user intended.
- **Open hardening.** Some hardening is in progress, including tightening file permissions on the root-owned OTA repository and building the Navigate-bearing automation safety gate. These are tracked in [roadmap.md](roadmap.md). Where an item is not yet fixed, this overview describes the area rather than a method.
