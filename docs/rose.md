# Rose: the intended architecture

Meridian runs on Debian's Linux kernel today. The intended long-term architecture replaces it with Rose, a from-scratch capability microkernel written in Rust, and moves Compass from a Linux daemon to a native layer whose authority the kernel enforces. Rose is in active development and is not yet usable as Meridian's kernel. Its own technical overview is at [Rose-Overview](https://github.com/TrashPanda2481/Rose-Overview).

This document covers Rose's status, the transition path, how Compass maps onto Rose, and the design of the adaptive layer.

## Status

| | |
|---|---|
| Language | Rust (`no_std` kernel), with a shared `abi` crate for kernel and userspace types |
| Boot | Limine, BIOS and UEFI |
| Test targets | QEMU smoke test on every change; Oracle VM VirtualBox before promotion to the stable branch |
| Kernel | Phase 0 primitives mostly done; Phase 1 (component model) in progress |
| Adaptive layer | Written design; no code |

Built and verified: physical frame allocator, GDT/IDT/TSS, kernel-owned page tables with W^X, kernel heap, PIC/PIT timer, a preemptive round-robin scheduler, per-task address spaces, ring 3, capabilities (CSpace with copy, mint, move, and cascading revoke), Untyped/Retype, Frame-Map, Endpoint IPC (Send, Receive, Call, Reply), and the boot handoff to a single root task.

Next: a loader and boot archive, so the root task can start components from real binaries and talk to them over IPC.

## Transition path

The plan is a hybrid. Full Linux syscall compatibility was rejected because matching Linux semantics that closely would make Rose Linux-shaped. A cold native replacement was rejected as a starting point because nothing would be usable until the whole native stack existed. Instead, a deliberately narrow compatibility shim bootstraps the transition, everything meant to be permanent is written native from the start, and the shim shrinks every phase.

| Phase | Scope | Status |
|---|---|---|
| 0 | Kernel primitives | Mostly done |
| 1 | Real component model: Untyped/Retype, IPC, root task, loader | In progress |
| 2 | Minimal native runtime: syscall wrappers without libc, the serial driver moved to userspace, virtio-blk and a minimal filesystem server | Not started |
| 3 | The shim: one userspace personality server covering a narrow slice of Linux syscalls, enough for a shell, coreutils, and a build toolchain | Not started |
| 4 | Replace Meridian's stack outward: init and service supervision, then the shell, then display, then applications, deleting shim coverage as each native piece lands | Not started |
| 5 | Shim at zero: native Meridian userland on Rose, no Linux-compatibility code | Not started |

## Where Compass fits

Compass on Meridian is the Linux-hosted predecessor of Rose's adaptive layer. Both follow the same rule: the setting that controls how much the system does on its own never grants authority. What changes on Rose is where that rule is enforced.

| | Meridian today | Rose (design) |
|---|---|---|
| What it observes | shell commands, window focus, and desktop switches, reported over the session D-Bus | per-component kernel counters, read through a scoped observe capability |
| How privileged work is authorized | polkit and pkexec to an allowlisted helper, administrator credential required | a capability minted by the consent broker from a specific user approval |
| How much runs automatically | the bearing, one setting for all of Compass (kernel updates always confirm) | per decision: handled once, remembered, or trained into a reflex; each is a separate revocable capability |
| Limits on the daemon | systemd sandbox directives in its unit file, which root can change | a bounded memory pool and registry limits enforced by the kernel, which the bounded component cannot raise |
| Withdrawing autonomy | lower the bearing | revoke the capability; the revoke cascades to everything derived from it |

## The adaptive layer

The full specification is in Rose-Overview's [adaptive.md](https://github.com/TrashPanda2481/Rose-Overview/blob/main/docs/adaptive.md). This section summarizes it.

### Not in the kernel

Rose does not run a model in the kernel. A reasoner in the kernel would grow the trusted base, add nondeterminism to paths with hard latency requirements, and put something that cannot be audited in the path that grants authority. The kernel stays reflexive: it enforces capabilities, isolation, and resource allocation. The adaptive layer runs as ordinary userspace components and holds only the capabilities it has been granted.

### Layers

| Layer | Component | Role |
|---|---|---|
| Sensing | Stem | Observe only. Keeps a baseline per signal, reports deviation, judges nothing. |
| Reflex | Rose kernel | Fast and non-optional. An untrainable floor, plus a tier that can be influenced within bounds. |
| Modulation | Resource manager | Biases allocation within ranges the kernel owns, and runs on its own inside limits granted once. A separate component from Compass, so it works in a build without the interface layer. |
| Intent | Compass and the user | Decides whether a deviation matters, proposes an action, and waits for consent. |

Two rules hold across the layers. Intent flows down and veto flows up: the untrainable floor overrides any policy or learned behavior. And the adaptive layer runs inside a resource bound that the kernel enforces and the layer cannot raise.

The whole layer is optional. A Rose build without Stem and Compass is a complete operating system.

### The untrainable floor

Kernel behavior that nothing can modify, bypass, or train around:

1. Capability and rights checks on every invocation.
2. Address-space isolation and W^X.
3. Fault containment: a fault takes down the offending component, not the system.
4. The authority-granting path: only the consent broker mints capabilities, and the adaptive layer cannot grant itself authority.
5. Revocation always completes.
6. The resource bound on the adaptive layer itself.
7. Hardware-safety cutoffs such as an overtemperature halt (planned).
8. A guaranteed minimum of CPU and memory for the path the user observes, consents, and revokes through (planned; needs scheduler reservations).

Items 1 to 6 map onto mechanisms the kernel already has: rights checks, paging with W^X, the fault path, the mint path, the capability derivation tree, and bounded Untyped memory.

### How authority is granted

A user approval is the only thing that gives Compass authority to act. Each approval takes one of three forms:

- **Handle once:** an ephemeral capability, used and discarded.
- **Remember:** a stored rule that mints the same capability again when the same situation recurs. From then on it runs in the modulation layer without asking.
- **Do not ask again:** a suppression entry.

Autonomy is therefore a set of delegated, revocable capabilities, each traceable to the approval that created it, rather than a global setting. A remembered decision that has proven itself can later be consolidated into a kernel reflex through a registration gate: a verified, capability-gated facility for fast-path rules, similar in shape to eBPF, whose verifier holds the untrainable floor as a hard limit.

The only learning signal is the user's own answers: approved, declined, or "do it differently." Nothing leaves the machine. The learner is rules plus a bounded case memory, not an opaque model, so every suggestion traces to a rule or a past decision.

### Grounding

Rose identifies what a component is from the capabilities it holds and the endpoints it talks to. A component that holds no display or input capability, and that no foreground component communicates with, is background work, and those facts are enforced by the kernel rather than inferred from process names. A component's self-declared manifest is read as a claim checked against that graph, and labels from the user add to it.

### Decision loops

Each adaptive behavior is a closed loop with five parts. The decomposition is settled; the concrete definitions are not written yet.

| Part | Question | Current direction |
|---|---|---|
| Signal | what Stem emits | per-component counters and sensor readings; a running mean and variance as the baseline; a sustained threshold crossing as a deviation (proposed) |
| State | the picture decisions are made against | the capability and endpoint graph, each node annotated with its baselines |
| Decision | whether a deviation matters and what to do | rules matching graph patterns plus deviations, and a case memory of past situations, actions, and verdicts (case-based reasoning) |
| Actuator | what the layer can do | a fixed, enumerated set: suspend, resume, priority bias, revoke, restart, cache hints; each is a kernel operation that requires a capability |
| Feedback | how a decision is judged | the user's verdict; whether a measured outcome may demote a remembered rule that fails to move its signal is open |

The closest engineering equivalent is an engine control unit: sensors (Stem), base tables (shipped rules), short-term closed-loop trim (modulation), long-term learned trim (remembered decisions and the case memory), the rev limiter and limp mode (the untrainable floor), a tuner with a laptop (the user, through Compass), and a verified reflash (the reflex gate).

### Example: one decision end to end

A sustained high CPU temperature. Steps marked "not built" depend on kernel or component work that has to exist first.

1. The scheduler accounts CPU time per component, and a thermal driver component reads the sensor. (not built)
2. Stem sees the temperature well above its baseline for a sustained period and notifies Compass, naming the largest contributor: a component using far more CPU than its own baseline. (notification objects not built)
3. Compass checks the graph. The component holds no display or input capability and nothing in the foreground talks to it, so it is background work. A rule matches (high temperature, background contributor) and proposes suspending it. The case memory has no prior verdict for this situation, so Compass asks.
4. The user approves and asks Compass to remember. The consent broker mints a suspend capability tagged with that approval and stores a rule that mints it again on the same trigger. (consent broker and badges not built)
5. The suspend runs, and the kernel checks the capability as it checks any other. Stem sees the temperature fall, and the outcome is recorded with the case. (thread suspend not built)
6. The next time the same situation occurs, the modulation layer acts without asking. Revoking the capability ends that autonomy immediately.

## Related work

The individual parts have prior art:

| Part | Prior art |
|---|---|
| Monitor, analyze, plan, execute loop | IBM autonomic computing (Kephart and Chess, 2003), the MAPE-K loop |
| Capability microkernel | KeyKOS, EROS, seL4, Fuchsia (Zircon) |
| Verified rules installed into a kernel fast path | eBPF; sched_ext (Linux 6.12) runs scheduling policy as BPF programs |
| Machine learning inside a kernel | LinnOS (OSDI 2020), a small neural network in the block layer predicting SSD latency; LAKE (ASPLOS 2023), GPU-accelerated in-kernel ML |
| Usage-driven resource management | Android Adaptive Battery, Windows SysMain, macOS App Nap, systemd-oomd with PSI |
| Rules plus case memory | expert systems and case-based reasoning |

What differs is the position of the adaptive component. In the systems above, the code that adapts the system is among the most privileged on the machine, and how much it does on its own is a global switch. In Rose it starts with no authority to act, each thing it may do without asking is a separate capability minted from one user approval and removable with one revoke, and its resource use is bounded by the kernel. Meridian approximates the same model on Linux with an unprivileged daemon and polkit, but there the guarantees are configuration that root can override. On Rose they are enforced by the kernel.

## Open questions

- The encoding of the observe capability in the kernel's rights set.
- Revocation semantics: whether a revoke always cascades down the derivation tree or capabilities can be sealed. The consent broker's three modes depend on the answer.
- The manifest schema, and how a manifest is checked against the capability graph.
- The case memory's size bound and eviction policy.
- The reflex gate's verifier: what a fast-path rule may express, and how the floor is proven as its limit.
- The concrete loop definitions above, and whether outcomes may demote a remembered rule.
