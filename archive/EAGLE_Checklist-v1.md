# EAGLE Project Checklist

*Archived 2026-11-22: the 6-Act plan from before the Phase A study, superseded by the ADRs in [`docs/adr/`](../docs/adr/README.md); [`EAGLE_Checklist.md`](../EAGLE_Checklist.md) maps each Act to the ADR that now owns it.*

> The flight computer is the project. The benchmark in Act III is an optional side experiment; if it holds up, it becomes the paper ([`RESEARCH_PAPER_PLAN.md`](RESEARCH_PAPER_PLAN.md)).

**Redundancy scope:** Acts I–V make the **sensors** triple-redundant: three IMUs, one MCU, one FPGA. The MCU and the FPGA are still single points of failure. Redundant compute is Act VI.

---

## Act I: The FPGA Reflex System & TMR
*The Cyclone II reads the sensors on a fixed schedule and votes on them.*
- [ ] **SPI masters:** Synthesizable SPI Mode 0 master in pure `ieee.numeric_std` with a configurable clock divider. One instance per IMU, so all three are sampled on the same clock edge, plus one for the barometer.
- [ ] **Mid-value voter:** Select the median of the three IMU readings, per axis. Exact majority voting does not work for sensors: three real IMUs never agree bit for bit.
- [ ] **Fault detector:** 3-bit fault vector that flags a channel which drifts past a threshold from the median or stops responding.
- [ ] **MCU link (SPI slave):** The MCU reads the voted data and the fault vector over SPI with DMA, at a fixed rate.
- [ ] **PWM generator:** Drive the airbrake servos and the buzzer.
- [ ] **Verification:** Self-checking GHDL testbench for each module, including a corrupted sensor and a dead one. GTKWave captures of the clock cycle where a faulty channel is isolated.
- [ ] **Timing closure:** `eagle.sdc` for the 50 MHz clock; positive setup/hold slack in Quartus TimeQuest.

## Act II: The RTOS Compute Core
*FreeRTOS and F´ on the microcontroller. Start this alongside Act I: getting F´ onto the target is the riskiest step in the project, and Act III needs it.*
- [ ] **Choose the MCU** (STM32 likely) and set up an ARM GCC toolchain (STM32Cube or plain CMake).
- [ ] **FreeRTOS:** Running on the MCU with preemptive multitasking.
- [ ] **F´ build:** Cross-compile NASA's F´ framework for the FreeRTOS target.
- [ ] **F´ heartbeat:** A basic F´ component that sends a serial heartbeat to the laptop, to prove the stack works.

## Act III: The Aerospace Bridge
*FPGA and MCU working together on the bench.*
- [ ] **Bench wiring:** FPGA to the three IMUs and the barometer, SPI link to the MCU. Everything is 3.3 V.
- [ ] **F´ link component:** A C++ F´ component that reads the FPGA's frames over SPI with DMA at a fixed rate.
- [ ] **Fault injection:** Disconnect or corrupt one IMU; confirm the fault vector flags it and the voted output stays valid.
- [ ] **Compliance audit:** Check my own C++ (not the F´ framework, which is a documented deviation) against MISRA C++:2023 and the Power of 10 rules, including no dynamic memory after initialization. Tooling: to be decided.
- [ ] **Benchmark (optional):** Compare the FPGA core against an MCU-only baseline. Method and fairness rules: [`RESEARCH_PAPER_PLAN.md`](RESEARCH_PAPER_PLAN.md).

## Act IV: State Machine & Telemetry
*Flight logic and data logging.*
- [ ] **SD card logging:** Integrate or write an SPI driver in F´ for continuous flight data logging to the microSD card.
- [ ] **Rocket state machine:** The flight states (IDLE, ARMED, ASCENT, APOGEE, DESCENT, TOUCHDOWN) as an F´ component.
- [ ] **Sub-GHz radio:** Wire a CC1101 radio to the MCU and write an F´ driver that sends CCSDS telemetry frames.
- [ ] **Flipper Zero ground station:** Receive and decode the telemetry frames on the Flipper's Sub-GHz radio.

## Act V: Guidance, Navigation, & Control
*Estimating the rocket's state and acting on it.*
- [ ] **Kalman filter:** An F´ component that fuses barometer and IMU data into an altitude and vertical-velocity estimate.
- [ ] **Airbrake PID:** A PID loop on the error between target and estimated apogee, commanding the FPGA's servo PWM.
- [ ] **Software-in-the-loop:** Feed simulated flight profiles (Python or MATLAB) into the F´ components on the laptop before any hardware test.

## Act VI: Advanced Aerospace Protocols
*Stretch goals, after the rest flies.*
- [ ] **Redundant compute:** Three MCUs run the same F´ deployment; the FPGA compares their outputs 2-out-of-3 (exact match). Needs the replicas kept in step and their outputs deterministic. This removes the MCU as a single point of failure.
- [ ] **Voter hardening:** The Cyclone II's SRAM configuration can itself be upset. Study mitigations such as TMR inside the voter and periodic reconfiguration.
- [ ] **CAN node:** A VHDL CAN controller for prioritized telemetry across the system.
- [ ] **SpaceWire node:** Only worth doing with a partner on the other end that speaks it; STM32s have no SpaceWire peripheral.
