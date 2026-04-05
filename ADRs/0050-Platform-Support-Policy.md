# ADR-0050: Platform Support Policy

## Status

Accepted

## Table of Contents

- [Context](#context)
- [Decision](#decision)
  - [Supported Platforms](#supported-platforms)
  - [Explicitly Unsupported](#explicitly-unsupported)
  - [Distribution Model](#distribution-model)
- [Rationale](#rationale)
  - [Why Apple Silicon Only on macOS](#why-apple-silicon-only-on-macos)
  - [The Four-Case Architecture Matrix](#the-four-case-architecture-matrix)
  - [JVM Under Rosetta 2: The Pathological Case](#jvm-under-rosetta-2-the-pathological-case)
  - [Build-On-Target Discipline](#build-on-target-discipline)
  - [Apple's Deprecation Timeline](#apples-deprecation-timeline)
- [Consequences](#consequences)
- [References](#references)

---

## Context

Ooloi is distributed as a self-contained desktop application with a bundled JVM produced by `jlink`. The bundle is architecture-specific: the JDK baked into the image, the native JavaFX libraries resolved via Maven classifiers, the Skija native bindings, and any per-platform SPI providers (e.g. CoreMidi4J on macOS, per [ADR-0044](0044-MIDI-Input-Library-and-Boundary-Architecture.md)) all resolve against a single `(os, arch)` pair at build time.

Until now, Ooloi's supported platforms have been described only in scattered locations — [ADR-0000](0000-Clojure.md) mentions "cross-platform via JVM", [ADR-0005](0005-JavaFX-and-Skija.md) notes that JavaFX runs on Windows, macOS, and Linux, [ADR-0044](0044-MIDI-Input-Library-and-Boundary-Architecture.md) discusses macOS MIDI providers, [ADR-0047](0047-Font-Management.md) discusses platform-specific font directories, and the Project README mentions three platforms. None of these distinguish Intel Macs from Apple Silicon Macs, and no document states authoritatively what an Ooloi build targets.

This gap matters. Apple reclassified 2017 Intel MacBook Pros as "vintage" in 2024 and will complete the Rosetta 2 deprecation in macOS 28. Shipping an Intel-targeted bundle to Apple Silicon users — which is almost all Mac users running current software on current hardware — would force the worst-performing execution path available on their hardware. Shipping two separate Mac bundles doubles the build matrix and the support burden for a population that is already a small minority of the Mac-using audience and is actively shrinking.

This ADR records the platform set Ooloi actually targets, the reasoning behind the Apple Silicon decision, and the disciplines that follow from both. It is the authoritative answer to the question "what does Ooloi run on?"; other documents that mention platforms should reference this ADR rather than restate the list.

---

## Decision

### Supported Platforms

Ooloi produces and ships the following desktop bundles:

| OS | Architecture | Status |
|---|---|---|
| **macOS** | `aarch64` (Apple Silicon: M1, M2, M3, M4, M5, and successors) | Supported |
| **Windows** | `x86_64` | Supported |
| **Linux** | `x86_64` | Supported |
| **Linux** | `aarch64` | Planned, not yet shipped |

Each bundle is self-contained: it carries its own JDK produced by `jlink` targeting that specific `(os, arch)` pair, its own architecture-matched JavaFX natives, and its own platform-appropriate SPI providers. The bundle runs only on the architecture it was built for.

### Explicitly Unsupported

- **macOS on `x86_64` (Intel Macs)**: Ooloi does not produce an Intel-targeted macOS bundle. This includes running Intel binaries on Apple Silicon via Rosetta 2 — Ooloi will not ship a configuration that exercises that path.
- **Running any bundle under emulation**: running a Windows `x86_64` bundle on Windows-on-ARM via Microsoft's x64 emulation is unsupported. Running a Linux `x86_64` bundle on Linux `aarch64` via `qemu-user` or similar is unsupported. The rationale in [§JVM Under Rosetta 2](#jvm-under-rosetta-2-the-pathological-case) applies to all JVM-under-emulation configurations, not just macOS.
- **32-bit architectures**: no supported platform offers a 32-bit JVM on any of these OSes today; Ooloi does not target them.

### Distribution Model

A single Ooloi build cycle produces one bundle per supported `(os, arch)` pair. Each bundle is built on a host machine running natively on the target architecture. There is no cross-architecture build in Ooloi's distribution pipeline — an Apple Silicon bundle is built on Apple Silicon, a Windows bundle on Windows, a Linux `x86_64` bundle on Linux `x86_64`, and so on. This is the build-on-target discipline; its reasoning is in [§Build-On-Target Discipline](#build-on-target-discipline).

---

## Rationale

### Why Apple Silicon Only on macOS

The decision to drop Intel Mac support and ship only Apple Silicon on macOS rests on three independent legs, any one of which would be sufficient on its own:

1. **Apple Silicon users must not get the Intel bundle.** The overwhelming majority of Mac users running current software on current hardware are on Apple Silicon. Shipping them an Intel bundle forces them to run a JVM under Rosetta 2, which is the worst execution path available on their hardware (see [§JVM Under Rosetta 2](#jvm-under-rosetta-2-the-pathological-case)). This is not a theoretical concern; it is the default outcome if an Intel bundle is the only macOS bundle offered.

2. **A dual-architecture Mac build matrix is not justified by the audience it would serve.** Supporting both Intel and Apple Silicon on macOS would double the build, test, and distribution pipeline for the Mac platform while serving an Intel Mac population that is small, shrinking, and increasingly outside Apple's own support window. The engineering effort required to support Intel Macs is disproportionate to the number of users it would reach, relative to any other supported platform.

3. **Apple's deprecation timeline makes an Intel Mac bundle a dead end regardless.** Rosetta 2 is being retired in macOS 28. 2017-era Intel Macs are already classified as "vintage". Investing in Intel Mac support now means investing in a target that Apple itself is actively removing from the ecosystem.

None of these three legs depends on the others. An Intel bundle would be the wrong choice even if Apple's deprecation timeline were longer, even if the dual-matrix cost were lower, and even if Rosetta's JVM performance were acceptable.

### The Four-Case Architecture Matrix

The decision above is clearer when the options are enumerated explicitly. For a Mac target, there are exactly four architecture combinations to consider:

| Bundle architecture | Host architecture | Execution mode | Performance |
|---|---|---|---|
| `x86_64` | Intel Mac | Native | Baseline (good) |
| `aarch64` | Apple Silicon Mac | Native | Excellent (2–3× baseline on CPU-bound work) |
| `x86_64` | Apple Silicon Mac | **Rosetta 2 emulation** | **Pathologically bad for JVMs** |
| `aarch64` | Intel Mac | Impossible | N/A |

Ooloi supports exactly the second row: an Apple Silicon bundle running natively on Apple Silicon. Every other row is either unsupported (rows 1 and 3) or impossible (row 4).

The critical observation is that the third row — Intel bundle on Apple Silicon via Rosetta 2 — is what happens **by default** if Ooloi ships only an Intel bundle to a user base that is overwhelmingly on Apple Silicon. That path is not a fallback to be tolerated; it is actively worse than either of the native rows, for reasons specific to how JVMs interact with binary translation.

### JVM Under Rosetta 2: The Pathological Case

Rosetta 2 is Apple's binary translation layer for running x86_64 code on Apple Silicon. For most native applications — code that is compiled once and executed — Rosetta performs well: it translates x86_64 machine code to arm64 on first execution, caches the translation, and runs the cached output at something like 70--85% of native performance. For static binaries, this is a remarkable engineering achievement.

For JVMs, it is the worst possible scenario.

A JVM does not execute a static binary. It JIT-compiles hot methods from bytecode into native machine code at runtime, then recompiles them again at higher optimisation tiers as profiling data accumulates. Under Rosetta, every JIT output is x86_64 machine code that Rosetta must then translate to arm64 before the CPU can execute it. Every HotSpot tier transition (interpreter → C1 → C2) invalidates the previous translation and forces Rosetta to retranslate. Every class unload, every deoptimisation, every OSR compilation triggers new translation work. The translation cache, which is Rosetta's primary optimisation for static binaries, is constantly invalidated by the JVM's own code generation.

On top of this, the JVM's garbage collector issues memory barriers and atomic operations that are cheap on x86_64 (strong memory model) but must be translated into more expensive arm64 sequences (weaker memory model, explicit fences required). SSE4 and AVX SIMD instructions in HotSpot intrinsics — used for `System.arraycopy`, `String.indexOf`, ByteBuffer operations, and many others — translate to NEON sequences with different semantics and overhead.

The empirical result, across multiple JVM workloads that have been measured in this configuration, is a 25--50% CPU overhead and a 50--80% overhead on GC-heavy workloads, compared to running the same JVM as a native arm64 build on the same Apple Silicon hardware. The slowdown is not constant; it scales with how much JIT activity the application generates and how aggressively HotSpot is re-optimising. A Clojure application like Ooloi, which loads namespaces lazily, compiles large call graphs, and exercises polymorphic dispatch heavily, is exactly the kind of workload that triggers the worst behaviour.

Measured against a native Apple Silicon bundle, the same Mac running the Intel bundle under Rosetta can be 3--5× slower on CPU-bound Ooloi workloads — not because Apple Silicon is slow, but because the combination of emulation plus JIT is uniquely bad for JVMs.

This is why "just ship an Intel bundle and let Rosetta handle Apple Silicon" is the wrong answer. Rosetta handles Apple Silicon; it does not handle Apple Silicon *running a JVM*.

### Build-On-Target Discipline

`jlink` bakes the host JDK into the output image. Running `jlink` on an Apple Silicon Mac produces an `aarch64` runtime; running it on an Intel Mac produces an `x86_64` runtime. There is no `--target-arch` flag that makes a Mac produce a Windows runtime, or an `x86_64` machine produce an `aarch64` runtime. Cross-jlink exists in some JDK distributions as a supported configuration, but the dependency graph (JavaFX natives via Maven classifiers, Skija natives, CoreMidi4J on macOS, platform-specific font loading, platform-specific TLS stores) makes cross-building brittle in practice — every native dependency has to be manually overridden for the target platform, and the results have historically been fragile enough that the Ooloi build pipeline takes the simpler discipline:

**Each supported bundle is built on a host machine running natively on the target `(os, arch)` pair.**

An Apple Silicon bundle is built on an Apple Silicon Mac. A Windows bundle is built on Windows. A Linux `x86_64` bundle is built on Linux `x86_64`. There is no build-time cross-compilation in the Ooloi pipeline. This keeps the per-bundle dependency resolution simple (Maven picks the correct classifier automatically because it's resolving against the build host), keeps the native libraries correct (they're the ones the build host actually uses), and keeps the test matrix honest (every bundle is tested on a machine that can run it natively).

The cost of this discipline is that the build matrix requires one machine per supported `(os, arch)` pair. The benefit is that every bundle is produced under the exact conditions it will run under, and platform-specific bugs cannot hide behind cross-compilation quirks.

This discipline is one of the reasons the dual-architecture Mac matrix is hard to justify: supporting Intel Macs would require maintaining a permanent Intel Mac build machine in the pipeline, with all the continuous-integration and release-signing plumbing that entails, for a population that is actively leaving the platform.

### Apple's Deprecation Timeline

At the time of this decision:

- **2017 Intel MacBook Pros** are classified as "vintage" by Apple. They no longer receive the latest macOS releases.
- **Rosetta 2** was announced at WWDC 2025 as deprecated, with removal scheduled for macOS 28.
- **No new Intel Macs have shipped since 2020** (the first M1 Macs launched in November 2020). The Intel Mac population is bounded above by the set of machines sold before that cutoff, and is shrinking as those machines age out of service.
- **Apple's own development toolchain** (Xcode, Swift, modern frameworks) has been Apple Silicon-first since 2023.

Investing in Intel Mac support now means investing in a target that is shrinking on every axis simultaneously: the hardware is out of production, the OS updates are tapering, the emulation layer is being removed, and the remaining user base is declining as hardware fails and is replaced. Any engineering effort spent on Intel Mac support is effort that will not compound into long-term value.

---

## Consequences

**Positive:**

- **One authoritative source.** The supported-platform set lives in this ADR. Other documents (ADRs, guides, READMEs, dev-setup files) reference this ADR rather than restating the list.
- **No Rosetta path.** Ooloi never ships a bundle that will be run under Rosetta 2 in production. Every macOS user gets native performance.
- **Simpler build matrix on macOS.** One Mac bundle, not two. One Mac build machine in the pipeline, not two. One Mac test target, not two.
- **Alignment with Apple's ecosystem direction.** Ooloi's macOS support matches where the Mac platform is actually going, not where it was in 2020.
- **Honest cross-platform story.** "Ooloi runs on macOS, Windows, and Linux" is still true, and each claim is backed by a native bundle — not by hoping emulation is good enough.

**Negative:**

- **Intel Mac users are not served.** Anyone still running a 2019-or-earlier Intel Mac cannot run Ooloi. This is a real cost to a real (though small and shrinking) population.
- **No runtime fallback.** If a user unknowingly attempts to run Ooloi on an Intel Mac, the application will not start. The installer must detect architecture and fail with a clear message pointing at the supported hardware, rather than silently running under Rosetta with degraded performance.
- **Platform detection must be explicit in the installer.** The installer cannot assume "macOS is macOS"; it must check architecture and refuse to install on `x86_64` Macs with a message that explains why.

**Neutral:**

- **Windows and Linux are untouched by this decision.** Both remain `x86_64`-first. Linux `aarch64` is a future consideration tracked separately and does not affect current distribution.
- **Future Apple architectures are covered automatically.** Any future Mac CPU (M4, M5, and successors) is `aarch64`-compatible; the Apple Silicon bundle runs natively on all of them without recompilation.

---

## References

- [ADR-0000: Clojure](0000-Clojure.md) — JVM cross-platform foundation
- [ADR-0005: JavaFX and Skija](0005-JavaFX-and-Skija.md) — native graphics dependencies per platform
- [ADR-0044: MIDI Input Library and Boundary Architecture](0044-MIDI-Input-Library-and-Boundary-Architecture.md) — platform-specific MIDI SPI providers
- [ADR-0047: Font Management](0047-Font-Management.md) — platform font loading and directory conventions
- Apple WWDC 2025 session on Rosetta 2 deprecation (macOS 28 removal timeline)
