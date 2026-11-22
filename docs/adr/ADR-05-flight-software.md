# ADR-05: Flight software stack

← [ADR index](../../README.md#decision-records)

| | |
|---|---|
| **Status** | Scoped |
| **Date** | 2026-11-22 |
| **Depends on** | [ADR-03](ADR-03-partitioning.md) (tiers, microcontroller), [ADR-02](ADR-02-fault-tolerance.md) (replica determinism of the commanded path) |
| **Decides** | where and how F´ runs, which RTOS runs under it (or none), how the commanded path is built so that two channels compute identical frames, and the MISRA C++:2023 and Power of 10 enforcement plan with its boundary |

## Context

F´ is the chosen framework: C++ components connected through typed ports, with the model written in FPP
and the C++ glue generated from it [1][13]. The workspace already has an F´ v4.3.0 scaffold
(`hello-project/`) and JPL's 2026 workshop material (`fprime_stuff/`). ADR-02 (Proposed) adds a
requirement F´ does not impose on its own: both channels must turn the same input frame into the same
command frame, bit for bit.

## Questions

1. Which tier runs F´, and in which deployment form: Linux, an RTOS, or no OS?
2. Which RTOS, if any, on the microcontroller: FreeRTOS, Zephyr, RTEMS, or F´ on bare metal?
3. How is the commanded path built so that it is deterministic across channels?
4. Where is the boundary for MISRA C++:2023 and the Power of 10, and which tools enforce what?

## What F´ provides (v4.3.0 documentation [1])

| Item | Fact |
|---|---|
| Component kinds | passive: no thread, no queue, runs in the caller's context; queued: a queue, no thread; active: a thread and a queue |
| Port kinds | sync, guarded (sync with a mutex), async (queued in the receiver) |
| Rate groups | `Svc.RateGroupDriver` divides one timer tick into rate groups; a group that has not finished when its next cycle starts has slipped, and F´ raises a WARNING_HI event |
| Cyclic work | F´'s own guidance: cyclic work with hard deadlines runs through sync ports on a rate group, because async work "would be run outside the rate group cycling context and thus slips and failure to reach deadlines would be hidden in another thread" |
| Health | `Svc.Health` pings active components and strokes a watchdog port only while every reply is in time (HTH-007): the heartbeat of ADR-04 |
| Memory | heap allocation only during initialization (`Fw::MemAllocator`); buffer pools at run time |
| No OS | "Applications can be written entirely as a set of passive components with one Timer component driving the entire application through passive rate groups" (baremetal guide) |
| Language | the framework builds as C++14 and C99 (`cmake/settings.cmake`); a project may set a newer standard for its own code |
| Ground | the F´ GDS: commands, telemetry and events from a laptop |

## Deployment options

| Option | Tier | F´ status | Notes |
|---|---|---|---|
| Linux | 1 (only if ADR-03 keeps it) | supported; Raspberry Pi reference built in F´ CI [1] | Ingenuity runs F´ on Linux on its Snapdragon [3] |
| FreeRTOS | 2 | `fprime-freertos`, a community OSAL port; its README says it "has so far been tested with Linux and ATSAMD51J19 (specifically, the Adafruit Feather M4)" and that much of the README is out of date [4]. One supported platform (Feather M4, F´ v4.3.0 [5]), no CI listed [1] | the maturity question the brief raised: thin |
| Zephyr | 2 | references for Pi Pico, Pi Pico 2 and Teensy 4.1, the last two built in F´ CI, plus PyCubed [1] | the best-supported RTOS path in F´ today; no Zephyr port for TMS570 or RM57 was found |
| RTEMS | 2 | no F´ reference project | RTEMS 6 has a TMS570LC4357 BSP, built big-endian with `-mbe32` [8]; ESA's RTEMS qualification data package covers SPARC LEON parts (GR712RC, GR740), not ARM [9] |
| F´ on bare metal | 2 | `fprime-baremetal` package; references "in development" for Pi Pico, Pi Pico 2, PyCubed [1]; JPL's 2026 platform slides list bare-metal F´ among Ingenuity's uses [2] | no scheduler: everything runs in rate-group order from one timer, so the commanded path is deterministic by construction; no active components |

JPL's February 2026 slides also list "STM32, Baremetal: in development" [2].

## RTOS comparison

| | FreeRTOS | Zephyr | RTEMS | none (F´ bare metal) |
|---|---|---|---|---|
| Licence | MIT [6] | Apache 2.0 | open source | F´ only (Apache 2.0) |
| Kernel size | "4000 to 9000 bytes" typical image [6] | configurable; not measured here | not measured here | none |
| Scheduling | preemptive priority; the kernel "never performs non-deterministic operations, such as walking a linked list, inside a critical section or interrupt" [6] | preemptive priority | preemptive priority, POSIX API | rate-group order |
| TMS570 / RM57 support | TI HALCoGen generates FreeRTOS projects (FreeRTOS 9.0.0 according to user reports; MPU-port problems reported) [10] | none found | `tms570lc4357_hdk` BSP [8] | needs only a timer, interrupts and drivers |
| Path to F´ | `fprime-freertos` (community) | `fprime-zephyr` (references with CI) | none | `fprime-baremetal` |
| Safety evidence | not checked here | IEC 61508 SIL 3 / SC 3 targeted, written concept approval obtained, certification not finished in the sources found [7] | ESA data package, SPARC LEON only [9] | n/a |

## Determinism of the commanded path (for ADR-02)

To be decided here, listed now so that the options above are judged against it:

- One chain of passive components on one rate group (sensor frame in, estimate, control, command frame
  out), started by the FPGA's frame (an interrupt or DMA completion); no threads, no queues, nothing that
  depends on the order in which interrupts arrive.
- Background work (telemetry, logging, ground commands) in active components at lower priority, handing
  data to the commanded path only at frame boundaries.
- The same binary and FPU configuration on both channels, and no value-changing optimisations such as
  `-ffast-math`.
- WCET by measurement on the target (cycle counter around the rate group, plus F´'s slip events): a
  measured maximum, which is not a proven WCET. Static WCET analysis is outside Phase A.

## MISRA C++:2023 and the Power of 10

What is known:

- MISRA C++:2023 was published in October 2023 and is written for C++17 (ISO/IEC 14882:2017) [11]. The
  document is sold as a single-user licence.
- F´ builds as C++14 by default, and F´ makes no MISRA claim: its flight-code clang-tidy configuration
  (`release.clang-tidy`) enables one check, `misc-no-recursion` (Power of 10 rule 1), and the whole-tree
  configuration enables six others, all modernisation and readability checks [1].
- A grep of the F´ v4.3.0 framework sources (`Fw`, `Svc`, `Os`, `Drv`, `Utils`, tests excluded) finds no
  `dynamic_cast`, `typeid`, `try` or `catch`, so `-fno-exceptions -fno-rtti` is likely to build. JPL's 2026
  platform-porting example toolchain uses `-fno-exceptions -pedantic -Wextra -Werror -Wold-style-cast`
  (with `-Wno-unused-parameter`) [2]. Confirm with a build (E5-2).
- Open tools already working in this workspace, from the spacetetris compliance run [15]: Cppcheck 2.10,
  whose MISRA addon checks MISRA C:2012 and partly reuses MISRA C++:2008 rules, with nothing for MISRA
  C++:2023 [12]; Cobra 5.4 with its `p10` and `jpl` rule sets for the Power of 10 and JPL's C standard,
  and a `C++/autosar` set its authors call "fast but not exhaustive" [14]. MISRA C++:2023 absorbed much of
  AUTOSAR C++14, so that set is a partial stand-in at best. GCC and Clang warnings; clang-tidy is not
  installed on the Pi.
- No open-source checker for MISRA C++:2023 was found. Full coverage needs a commercial tool.
- The brief expects F´'s generated FPP glue to "trip MISRA C++:2023 heavily". That is plausible and
  unmeasured (E5-1).

Boundary, as an option to decide:

| Code | Rules | Evidence |
|---|---|---|
| Hand-written flight-critical components: estimator, control, modes, fault manager, FPGA link driver | MISRA C++:2023 and the Power of 10 | a compliance record with deviations, in the form of spacetetris's `COMPLIANCE.md` |
| F´ framework and FPP-generated code | one documented deviation: built as C++14 by default, not written to MISRA | the deviation entry |
| Ground and test code | none | – |

Until a checker covers MISRA C++:2023, the code *follows* it; "compliant" needs the checker run and the
deviation record (README rules of this workspace).

Candidate flags for EAGLE's own components: `-std=c++17` (if F´ permits it for project code, untested),
`-Wall -Wextra -Werror -pedantic -fno-exceptions -fno-rtti -Wold-style-cast -Wconversion -Wsign-conversion
-Wshadow`, then tuned the way spacetetris's `gcc-strict.flags` was.

## Evidence that would settle this ADR

| Id | Evidence |
|---|---|
| E5-1 | Generate one F´ component and a small topology in the `hello-project` sandbox (not in EAGLE); run Cppcheck, Cobra (`p10`, `jpl`, `C++/autosar`) and clang-tidy; count findings in generated against hand-written code |
| E5-2 | Build the F´ framework on Linux with `-fno-exceptions -fno-rtti` and the strict warning set |
| E5-3 | Port spike: a minimal F´ deployment on each candidate MCU (TMS570 or RM57 through FreeRTOS or bare metal, S32K344, and RP2350 through Zephyr as the known-good control) |
| E5-4 | WCET method on target: cycle-counter distribution and observed maximum of the commanded rate group |
| E5-5 | Determinism: two channels fed the same recorded frames for 10⁶ frames, outputs compared bit for bit |

## Sources

1. F´ v4.3.0 documentation and sources: `docs/user-manual/overview/03-port-comp-top.md`,
   `framework/component-and-port-selection.md`, `design-patterns/rate-group.md`,
   `framework/run-baremetal.md`, `framework/supported-platforms.md`, `framework/memory-management/`,
   `Svc/Health/docs/sdd.md`, `cmake/settings.cmake`, `release.clang-tidy`, `.clang-tidy`.
   <https://fprime.jpl.nasa.gov/latest/docs/user-manual/>
2. K. F. Ortega, "F Prime Platforms," JPL Advanced F´ Workshop, February 2026.
3. E. Ackerman, interview with T. Canham (JPL), IEEE Spectrum, 2021.
   <https://spectrum.ieee.org/automaton/aerospace/robotic-exploration/nasa-designed-perseverance-helicopter-rover-fly-autonomously-mars>
4. fprime-community, `fprime-freertos` README. <https://github.com/fprime-community/fprime-freertos>
5. fprime-community, `fprime-featherm4-freertos-reference` README (F´ v4.3.0).
   <https://github.com/fprime-community/fprime-featherm4-freertos-reference>
6. AWS, "FreeRTOS kernel fundamentals." <https://docs.aws.amazon.com/freertos/latest/userguide/dev-guide-freertos-kernel.html>
7. Zephyr Project, "Zephyr Safety Overview" (SIL 3 / SC 3 target, Route 3s); Linux Foundation release on
   the IEC 61508 concept approval. <https://docs.zephyrproject.org/latest/safety/safety_overview.html>
8. RTEMS 6.2 User Manual, BSPs, "arm/tms570"; `bsps/arm/tms570/README` (link with `-mbe32`).
   <https://docs.rtems.org/docs/6.2/user/bsps/arm/tms570.html>
9. ESA, RTEMS SMP Qualification Data Package. <https://rtems-qual.io.esa.int/>
10. TI, SPNA237 (FreeRTOS on Hercules, MPU regions) <https://www.ti.com/lit/pdf/spna237>; TI E2E user
    reports on HALCoGen's FreeRTOS 9.0.0.
11. MISRA, "MISRA C++:2023" product page. <https://misra.org.uk/product/misra-cpp2023/>
12. Cppcheck 2.10, `addons/misra.py` header ("MISRA C 2012 checkers, Partially reused for MISRA C++
    2008 checking"), Debian package.
13. R. Bocchino, T. Canham, G. Watney, L. Reder, J. Levison, "F Prime: An Open-Source Framework for
    Small-Scale Flight Software Systems," Small Satellite Conference, 2018, Session 12.
    <https://digitalcommons.usu.edu/smallsat/2018/all2018/328>
14. G. J. Holzmann, Cobra, `rules/main/p10.cobra`, `rules/main/jpl.cobra`, `rules/C++/autosar.cobra`.
    <https://github.com/nimble-code/Cobra>; and "The Power of 10: Rules for Developing Safety-Critical
    Code," IEEE Computer 39(6):95–99, 2006, DOI 10.1109/MC.2006.212.
15. spacetetris, `COMPLIANCE.md` (MISRA C:2012 and Power of 10 record, tools and commands).
    <https://github.com/spaceadler/spacetetris>

## History

| Date | Change |
|---|---|
| 2026-11-22 | Scoped |
