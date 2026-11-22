# EAGLE Checklist

> Phase A: the decisions in order and the evidence each one is waiting for. No hardware is bought and no
> code is written in this phase. The roadmap's "EAGLE Act I" to "Act VI" items link here; the last table
> says where each old Act went.

## Decisions

A box is ticked when the owner accepts the ADR ([ADR-00](docs/adr/ADR-00-process.md) defines the statuses).

- [ ] [ADR-00](docs/adr/ADR-00-process.md): how decisions are recorded *(Proposed 2026-11-22)*
- [ ] [ADR-01](docs/adr/ADR-01-mission-profile.md): vehicle, validation path, feature scope *(drafted, Proposed 2026-11-22; seven questions for the owner)*
- [ ] [ADR-02](docs/adr/ADR-02-fault-tolerance.md): fault model, fault-tolerance pattern, degradation and recovery *(drafted, Proposed 2026-11-22; seven questions for the owner)*
- [ ] [ADR-03](docs/adr/ADR-03-partitioning.md): tiers, function map, FPGA budget, microcontroller *(Scoped 2026-11-22)*
- [ ] [ADR-04](docs/adr/ADR-04-interchip-icd.md): inter-chip links and ICD *(Scoped 2026-11-22)*
- [ ] [ADR-05](docs/adr/ADR-05-flight-software.md): F´ deployment, RTOS, MISRA C++:2023 and Power of 10 boundary *(Scoped 2026-11-22)*
- [ ] [ADR-06](docs/adr/ADR-06-hdl-verification.md): HDL, simulation, formal checks, SITL *(Scoped 2026-11-22)*

## Evidence that needs only documents

- [ ] Rate-loop crossover and phase margin of the candidate airframe, from a published identification or
      the vendor, to replace the assumed 10 to 20 Hz and 45° ([ADR-01](docs/adr/ADR-01-mission-profile.md) §1)
- [ ] E3-2: pin count of the EP2C5 board from its schematic, against the pin list in ADR-04
- [ ] E4-5: nERROR latency from TI's documentation for the TMS570 or RM57
- [ ] E4-6: input thresholds of the chosen microcontroller, to size the reset pull-down
- [ ] E6-1, first half: the VHDL-2008 and SystemVerilog support lists in the Quartus II 13.0 handbook

## Evidence that needs a board, a tool install or sandbox code

Each waits until the ADR it serves is Accepted. Sandbox code goes in `hello-project/`, never in EAGLE.

- [ ] E3-1: Quartus fitter reports per block; E3-3: F´ port spike per microcontroller; E3-4: the FPGA
      versus MCU sampling benchmark (from the archived paper plan); E3-5: CRC_ERROR pass time
- [ ] E4-1 to E4-4: frame-engine simulation, link timing capture, IMU sample spread on a shared clock,
      ESC behaviour on frame loss
- [ ] E5-1 to E5-5: MISRA findings in generated against hand-written F´ code, a strict-flag F´ build,
      the port spike, WCET method, bit-for-bit determinism over 10⁶ frames
- [ ] E6-1 second half to E6-4: construct tests in Quartus, simulator run times on the Pi, the formal
      flow on the laptop, SITL at 500 Hz

## After Phase A

Not started. The implementation phases (supervisor RTL and its proofs, F´ on the chosen microcontroller,
the HIL bench, the rig, then free flight) are planned from the Accepted ADRs; the archived Acts are a
starting list, not the plan.

## Where the old Acts went

| Roadmap item | Old content ([`archive/EAGLE_Checklist-v1.md`](archive/EAGLE_Checklist-v1.md)) | Now owned by |
|---|---|---|
| Act I: The FPGA Reflex System & TMR | SPI masters, mid-value voter, fault detector, MCU link, PWM, GHDL tests, timing closure | ADR-02 (pattern), ADR-03 (what the FPGA does, budget), ADR-04 (links), ADR-06 (HDL, verification) |
| Act II: The RTOS Compute Core | MCU choice, FreeRTOS, F´ build, heartbeat | ADR-03 (microcontroller), ADR-05 (RTOS, F´; the heartbeat through `Svc.Health`) |
| Act III: The Aerospace Bridge | bench wiring, F´ link component, fault injection, compliance audit, optional benchmark | ADR-04 (ICD), ADR-05 (compliance boundary), ADR-06 (fault injection, HIL), ADR-03 evidence E3-4 (benchmark) |
| Act IV: State Machine & Telemetry | SD logging, rocket flight states, CC1101 radio, Flipper Zero ground station | ADR-01 (logging, ground link, the rocket as alternative profile), ADR-02 (degradation state machine) |
| Act V: Guidance, Navigation, & Control | Kalman filter, airbrake PID, SITL | ADR-01 (the vehicle decides the GNC), ADR-06 (SITL) |
| Act VI: Advanced Aerospace Protocols | three MCUs voted 2-out-of-3, voter hardening against SRAM upsets, CAN node, SpaceWire node | ADR-02 (2-out-of-3 is pattern P2; configuration upsets are fault F7), ADR-03 (flash FPGA option), ADR-04 (CAN options; SpaceWire dropped) |
