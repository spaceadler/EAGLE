# EAGLE Checklist

> Phase A: the decisions in order and the evidence each one is waiting for. The vehicle is a sounding
> rocket and the flight computer is verified as a FlatSat. No hardware is bought and no code is written in
> this phase. The roadmap's "EAGLE Act I" to "Act VI" items link here; the last table says where each old
> Act went.

## Decisions

A box is ticked when the owner accepts the whole ADR ([ADR-00](docs/adr/ADR-00-process.md) defines the
statuses).

- [ ] [ADR-00](docs/adr/ADR-00-process.md): how decisions are recorded *(Proposed 2026-11-22; two questions for the owner)*
- [ ] [ADR-01](docs/adr/ADR-01-mission-profile.md): vehicle, test approach, flight phases, sensors, actuators *(Accepted in part 2026-11-22: sounding rocket, FlatSat, no camera; eight scope questions for the owner)*
- [ ] [ADR-02](docs/adr/ADR-02-fault-tolerance.md): fault model, pattern, who may fire, degradation and recovery *(Proposed, re-derived for the rocket 2026-11-22; eight questions for the owner)*
- [ ] [ADR-03](docs/adr/ADR-03-partitioning.md): function map, FPGA budget, whether the EP2C5 flies, microcontroller *(Scoped, rocket 2026-11-22)*
- [ ] [ADR-04](docs/adr/ADR-04-interchip-icd.md): inter-chip links, frame period, supervisor and pyro lines *(Scoped, rocket 2026-11-23)*
- [ ] [ADR-05](docs/adr/ADR-05-flight-software.md): F´ deployment, RTOS, phase detection, ground commands, MISRA C++:2023 and Power of 10 boundary *(Scoped, rocket 2026-11-23)*
- [ ] [ADR-06](docs/adr/ADR-06-hdl-verification.md): HDL, simulation, formal checks, SITL, the FlatSat *(Scoped, rocket 2026-11-23)*

## Evidence that needs only documents

- [ ] Motor and airframe data for the chosen target apogee, to replace ADR-01's §2 assumptions (boost
      acceleration, drag, deployment speeds)
- [ ] E3-2: pin count of the EP2C5 board from its schematic, against ADR-04's signal count (51 to 66 for P1)
- [ ] E4-5: nERROR latency from TI's documentation for the TMS570 or RM57
- [ ] E4-6: input thresholds, reset pin pull-up and GPIO state during reset of the chosen microcontroller
- [ ] E4-7, first half: the chosen microcontroller's absolute maximum input ratings (back-powering)
- [ ] E4-8: the barometer's conversion schedule at the chosen oversampling against the frame period
- [ ] E6-1, first half: the VHDL-2008 and SystemVerilog support lists in the Quartus II 13.0 handbook
- [ ] E6-7: error-forcing and self-test features of the chosen lockstep microcontroller

## Evidence that needs a board, a tool install or sandbox code

Each waits until the ADR it serves is Accepted. Sandbox code goes in `hello-project/`, never in EAGLE.

- [ ] E3-1: Quartus fitter reports per block; E3-3: F´ port spike per microcontroller; E3-4: the FPGA
      versus MCU sampling benchmark (from the archived paper plan), run on the FlatSat; E3-5: CRC_ERROR
      pass time
- [ ] E4-1 to E4-4: frame-engine simulation, link timing on the FlatSat, IMU sample spread on a shared
      clock, every supervisor and pyro line held at its safe level through power-up, reconfiguration and
      a channel power cycle; E4-7, second half: a power cycle resets a channel while the FPGA drives its link
- [ ] E5-1 to E5-6: MISRA findings in generated against hand-written F´ code, a strict-flag F´ build, the
      port spike, WCET method, bit-for-bit determinism over 10⁶ frames, phase machine and interlock flown
      against RocketPy or OpenRocket profiles
- [ ] E6-1, second half, to E6-6: construct tests in Quartus, simulator run times on the Pi, the formal flow
      on the laptop, SITL at 100 Hz, a sensor-emulator prototype
- [ ] E6-8: one unattended failover campaign on the FlatSat, 299 trials of one path

## After Phase A

Not started. The implementation phases (supervisor RTL and its proofs, F´ on the chosen microcontroller,
the emulated FlatSat, the hardware FlatSat and its failover campaigns, then flight) are planned from the
Accepted ADRs; the archived Acts are a starting list, not the plan.

## Where the old Acts went

| Roadmap item | Old content ([`archive/EAGLE_Checklist-v1.md`](archive/EAGLE_Checklist-v1.md)) | Now owned by |
|---|---|---|
| Act I: The FPGA Reflex System & TMR | SPI masters, mid-value voter, fault detector, MCU link, PWM, GHDL tests, timing closure | ADR-02 (pattern, fire interlock), ADR-03 (what the FPGA does, budget), ADR-04 (links, pyro lines), ADR-06 (HDL, verification) |
| Act II: The RTOS Compute Core | MCU choice, FreeRTOS, F´ build, heartbeat | ADR-03 (microcontroller), ADR-05 (RTOS, F´; the heartbeat through `Svc.Health`) |
| Act III: The Aerospace Bridge | bench wiring, F´ link component, fault injection, compliance audit, optional benchmark | ADR-04 (ICD), ADR-05 (compliance boundary), ADR-06 (the FlatSat, fault injection), ADR-03 evidence E3-4 (benchmark) |
| Act IV: State Machine & Telemetry | SD logging, rocket flight states, CC1101 radio, Flipper Zero ground station | ADR-01 (flight phases, logging, ground link; the rocket is now the vehicle), ADR-02 (degradation state machine), ADR-05 (phase detection, ground commands) |
| Act V: Guidance, Navigation, & Control | Kalman filter, airbrake PID, SITL | ADR-01 (airbrakes optional), ADR-05 (the estimator in the commanded path), ADR-06 (SITL on flight profiles) |
| Act VI: Advanced Aerospace Protocols | three MCUs voted 2-out-of-3, voter hardening against SRAM upsets, CAN node, SpaceWire node | ADR-02 (2-out-of-3 is pattern P2; configuration upsets are fault F7), ADR-03 (flash FPGA option), ADR-04 (classical CAN reassessed at 100 Hz; SpaceWire dropped) |
