# EAGLE Research Paper Plan & Benchmark Checklist

> **Phase 1 of Project EAGLE:** Scientific validation and empirical benchmarking of the FPGA reflex core before full airframe avionics integration.

---

## 1. Executive Summary & Academic Framing

### The Research Question
> *In resource-constrained, low-cost New Space avionics (CubeSats, amateur suborbital rocketry), what is the quantifiable reduction in timing jitter, Worst-Case Execution Time (WCET) variance, and MCU processing overhead when Triple Modular Redundant (TMR) sensor ingestion and voting are offloaded to dedicated FPGA fabric versus standard RTOS software polling?*

### The Core Hypothesis
Offloading concurrent 3x SPI sensor acquisition, hardware median voting, and fault isolation to a deterministic VHDL pipeline on low-cost FPGA fabric (`Cyclone II EP2C5T144C8`) eliminates RTOS interrupt latency jitter (reducing execution variance from $\pm 50\,\mu\text{s}$ to $< 50\,\text{ns}$) while reclaiming over 20% of MCU cycles for Guidance, Navigation, and Control (GNC) algorithms.

### Target Venues
- **IEEE DASC (Digital Avionics Systems Conference)** — Premier avionics track.
- **ESA/DLR PAC Symposium (European Rocket and Balloon Programmes)** — Core Bavarian/European target.
- **IEEE Aerospace Conference** — Big Sky student track.

---

## 2. Experimental Testbench (The Vertical Slice)

The experiment does not require building an entire rocket airframe. It isolates a strict hardware-in-the-loop bench:

```text
 ┌────────────────────────────────────────────────────────┐
 │                   EXPERIMENTAL BENCH                   │
 ├──────────────────────────┬─────────────────────────────┤
 │ BASELINE (Software Only) │ PROPOSED (HW-SW Co-Design)  │
 ├──────────────────────────┼─────────────────────────────┤
 │ STM32 / FreeRTOS         │ STM32 / FreeRTOS            │
 │   │ (SPI ISRs)           │   │ (Single UART / DMA Stream)│
 │   ▼                      │   ▲                         │
 │ 3x SPI Polling Tasks     │ Cyclone II FPGA (VHDL)      │
 │   ▼                      │   ├── 3x Concurrent SPI     │
 │ Software Median Voter(C) │   ├── 1-Cycle TMR Voter     │
 │   ▼                      │   └── Fault Status Injector │
 │ Actuator PWM Out         │   ▼                         │
 │                          │ 3x ICM-42688-P IMUs (SPI)   │
 └──────────────────────────┴─────────────────────────────┘
```

### Metrics to Measure & Graph (The Paper's Figures)
1. **Jitter Histogram:** Distribution of loop turnaround latency over $N = 100{,}000$ cycles.
2. **WCET Upper Bound:** Hard maximum latency spike under high bus load.
3. **MCU CPU Utilization:** Profiler delta between RTOS SPI interrupt servicing vs. passive DMA frame ingest.
4. **Fault Recovery Latency:** Microseconds between simulated sensor corruption and voter fault flag assertion.

---

## 3. Phase 1 Checklist: The Research Paper Execution

### Step 1: FPGA RTL Implementation (VHDL / DO-254 Discipline)
- [ ] **`spi_master.vhd`**: Write a synthesizable SPI Mode 0 master in pure `ieee.numeric_std` with configurable clock divider.
- [ ] **`tmr_voter.vhd`**: Implement single-cycle median voting across three 16-bit signed sensor streams with a configurable error threshold ($G\_MAX\_DIFF$).
- [ ] **`fault_detector.vhd`**: Emit a 3-bit active fault vector (`o_fault_vec`) flagging drifting or dead sensor channels.
- [ ] **`telemetry_tx.vhd`**: Packetize consensus sensor data + fault status byte into an 8N1 serial telemetry stream.
- [ ] **`eagle_reflex_top.vhd`**: Top-level Quartus project wiring the 3 SPI instances, voter, and UART transmitter.

### Step 2: Verification & Simulation (Waveform Generation)
- [ ] **`tb_tmr_voter.vhd`**: GHDL self-checking testbench asserting nominal consensus and single-event sensor corruption.
- [ ] **Waveform Plots**: Generate GTKWave VCD waveforms capturing the exact nanosecond where a faulty sensor is isolated (Paper Figure 3).
- [ ] **Timing Closure**: Write `eagle.sdc` for 50 MHz clock constraints; verify positive setup/hold slack in Quartus TimeQuest.

### Step 3: Hardware Benchtop & Data Collection
- [ ] **Wire Hardware Bench**: Wire Cyclone II board to 3x SPI IMUs (ICM-42688-P; the MPU-6050 first named here is I2C-only) and connect UART to MCU. Wiring table: [`bench/README.md`](bench/README.md).
- [ ] **FreeRTOS Baseline Firmware**: Program baseline FreeRTOS firmware polling the 3 sensors directly; log timing timestamps.
- [ ] **Co-Design Firmware**: Program firmware receiving FPGA telemetry packets via DMA; log timing timestamps.
- [ ] **Oscilloscope / Logic Analyzer Captures**: Capture physical bus jitter waveforms on GPIO toggle pins.
- [ ] **Data Processing**: Generate latency distribution histograms and CPU overhead plots in Python (`matplotlib`).

### Step 4: Paper Writing & Submission
- [ ] **Draft Abstract & Introduction**: Frame the COTS reliability challenge for sounding rockets.
- [ ] **System Architecture Section**: Document the VHDL pipeline and hardware-software partition.
- [ ] **Evaluation Section**: Embed the latency histograms, WCET comparisons, and fault recovery waveforms.
- [ ] **Faculty Review**: Review draft with GJU Robotics & Autonomous Systems Hub professors for lab co-authorship.
- [ ] **Conference Submission**: Submit to target conference (IEEE DASC / ESA PAC).

---

## 4. Phase 2: System Implementation (Post-Paper Transition)

Once the research paper is drafted and verified on hardware, the reflex core is **proven silicon**. You then proceed with full rocket integration across the remaining Acts in [`EAGLE_Checklist.md`](EAGLE_Checklist.md):
- Integrate NASA F´ framework components on the compute MCU.
- Implement flight state machine (IDLE $\rightarrow$ ARMED $\rightarrow$ ASCENT $\rightarrow$ APOGEE $\rightarrow$ TOUCHDOWN).
- Connect physical airbrake servos and calibrate the FPGA PWM driver.
- Deploy Sub-GHz telemetry link to the Flipper Zero ground station.
