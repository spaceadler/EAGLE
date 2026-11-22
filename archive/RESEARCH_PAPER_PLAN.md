# EAGLE Research Paper Plan & Benchmark Checklist

*Archived 2026-11-22: an implementation-phase experiment, paused while Phase A runs; its research question, fairness rules and metrics carry over as evidence item E3-4 of [ADR-03](../docs/adr/ADR-03-partitioning.md).*

> **An optional side experiment of EAGLE (Act III):** benchmark the FPGA reflex core against the same job done in MCU software. The flight computer does not depend on it.

---

## 1. Executive Summary & Academic Framing

### The Research Question
> *In low-cost avionics (CubeSats, amateur suborbital rocketry), what changes in sample-time jitter, end-to-end latency and MCU load when triple-redundant (TMR) sensor reading and voting move from MCU software to FPGA fabric?*

### The Hypothesis
Sampling and voting on the FPGA (`Cyclone II EP2C5T144C8`) should bring sample-time jitter down to the order of one clock period (20 ns at 50 MHz) and take SPI servicing off the MCU. The open question is how much of that survives end to end, since the control loop still runs on the MCU under FreeRTOS. The benchmark measures both instead of assuming either.

### Fair Baseline
The comparison only means something if the baseline is a good software design, not a slow one:
- Same MCU, same FreeRTOS + F´ build, same sample rate.
- The baseline reads the IMUs over SPI **with DMA** and votes in C (median select), as a careful engineer would write it.
- The only difference between the two builds is where the sensors are read and voted.

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
 │ STM32 / FreeRTOS + F´    │ STM32 / FreeRTOS + F´       │
 │   │ (SPI + DMA)          │   │ (SPI link + DMA)        │
 │   ▼                      │   ▲                         │
 │ 3x SPI Sensor Reads      │ Cyclone II FPGA (VHDL)      │
 │   ▼                      │   ├── 3x Concurrent SPI     │
 │ Software Median Voter(C) │   ├── 1-Cycle TMR Voter     │
 │   ▼                      │   └── Fault Status Injector │
 │ Actuator PWM Out         │   ▼                         │
 │                          │ 3x ICM-42688-P IMUs (SPI)   │
 └──────────────────────────┴─────────────────────────────┘
```

### Metrics to Measure & Graph (The Paper's Figures)
1. **Sample-time jitter:** Spread of the moment each sensor is actually sampled (GPIO toggle at the sample instant), over $N = 100{,}000$ cycles.
2. **End-to-end latency:** Sensor sample to actuator command, as a histogram plus its observed maximum under high bus load. This is a measured maximum, not a proven WCET.
3. **MCU CPU load:** Same build, same rate, measured with an idle-task counter.
4. **Fault detection latency:** Time from injected sensor corruption to the fault flag, in clock cycles on the FPGA and in microseconds in the baseline.

---

## 3. Benchmark Checklist

Steps 1–2 are the same work as Act I of [`EAGLE_Checklist-v1.md`](EAGLE_Checklist-v1.md); they are repeated here so the paper's method is complete.

### Step 1: FPGA RTL Implementation (VHDL)
- [ ] **`spi_master.vhd`**: Write a synthesizable SPI Mode 0 master in pure `ieee.numeric_std` with configurable clock divider.
- [ ] **`tmr_voter.vhd`**: Implement single-cycle median voting across three 16-bit signed sensor streams with a configurable error threshold ($G\_MAX\_DIFF$).
- [ ] **`fault_detector.vhd`**: Emit a 3-bit active fault vector (`o_fault_vec`) flagging drifting or dead sensor channels.
- [ ] **MCU link (SPI slave)**: Expose the voted sensor data + fault status to the MCU, read with DMA.
- [ ] **`eagle_reflex_top.vhd`**: Top-level Quartus project wiring the 3 SPI instances, the voter and the MCU link.

### Step 2: Verification & Simulation (Waveform Generation)
- [ ] **`tb_tmr_voter.vhd`**: GHDL self-checking testbench asserting nominal consensus, a corrupted sensor and a dead one.
- [ ] **Waveform Plots**: Generate GTKWave VCD waveforms capturing the clock cycle where a faulty sensor is isolated (Paper Figure 3).
- [ ] **Timing Closure**: Write `eagle.sdc` for 50 MHz clock constraints; verify positive setup/hold slack in Quartus TimeQuest.

### Step 3: Hardware Benchtop & Data Collection
- [ ] **Wire Hardware Bench**: Wire Cyclone II board to 3x SPI IMUs (ICM-42688-P; the MPU-6050 first named here is I2C-only) and connect the SPI link to the MCU. Wiring table: [`phase-1-folders.md`](phase-1-folders.md#bench).
- [ ] **Baseline Firmware**: FreeRTOS + F´ reading the 3 sensors over SPI with DMA and voting in C; log timing timestamps.
- [ ] **Co-Design Firmware**: The same build reading the FPGA's voted frames over SPI with DMA; log timing timestamps.
- [ ] **Oscilloscope / Logic Analyzer Captures**: Capture physical bus jitter waveforms on GPIO toggle pins.
- [ ] **Data Processing**: Generate latency distribution histograms and CPU overhead plots in Python (`matplotlib`).

### Step 4: Paper Writing & Submission
- [ ] **Draft Abstract & Introduction**: Frame the COTS reliability challenge for sounding rockets.
- [ ] **System Architecture Section**: Document the VHDL pipeline and hardware-software partition.
- [ ] **Evaluation Section**: Embed the latency histograms, WCET comparisons, and fault recovery waveforms.
- [ ] **Faculty Review**: Review draft with the university lab's faculty for co-authorship.
- [ ] **Conference Submission**: Submit to target conference (IEEE DASC / ESA PAC).

---

## 4. Relationship to the Flight Computer

The benchmark reuses the Act I core and the Act II firmware; it adds a baseline build and the measurements. The flight computer (Acts IV–VI in [`EAGLE_Checklist-v1.md`](EAGLE_Checklist-v1.md)) does not wait for it, and a negative result (the FPGA's advantage disappears end to end) is still a result worth writing up.
