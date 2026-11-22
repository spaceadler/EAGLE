# Phase 1 folder layout

*Archived 2026-11-22: the READMEs of the seven empty Phase 1 folders (`rtl/ tb/ quartus/ fw/ bench/ analysis/ paper/`), merged here as they were (links updated); Phase A decides what the implementation folders will be.*

## rtl

Synthesizable VHDL for the reflex core, in pure `ieee.numeric_std`. From [`EAGLE_Checklist-v1.md`](EAGLE_Checklist-v1.md), Act I:

| File | Role |
|---|---|
| `spi_master.vhd` | SPI Mode 0 master with a configurable clock divider, one per IMU plus one for the barometer |
| `tmr_voter.vhd` | Single-cycle median vote across three 16-bit signed streams, threshold `G_MAX_DIFF` |
| `fault_detector.vhd` | 3-bit fault vector `o_fault_vec` for drifting or dead channels |
| MCU link | SPI slave: the MCU reads the voted data and the fault vector with DMA |
| PWM | Airbrake servos and buzzer |
| `eagle_reflex_top.vhd` | Top level: the SPI masters, the voter, the MCU link and the PWM |

## tb

GHDL testbenches. From the plan, step 2: `tb_tmr_voter.vhd`, a self-checking testbench for nominal consensus and single-event sensor corruption, with GTKWave VCDs capturing the moment a faulty sensor is isolated (paper figure 3).

## quartus

The Quartus project for the `EP2C5T144C8` (Quartus 13.0sp1, the last release with Cyclone II). From the plan, step 2: `eagle.sdc` with the 50 MHz clock constraint, and positive setup/hold slack in TimeQuest.

## fw

MCU firmware: FreeRTOS + F´ ([`EAGLE_Checklist-v1.md`](EAGLE_Checklist-v1.md), Acts II–III). For the optional benchmark, two builds that differ only in where the sensors are read and voted:

| Build | What it does |
|---|---|
| Co-design | Reads the FPGA's voted frames over SPI with DMA and logs timestamps |
| Baseline | Reads the 3 sensors itself over SPI with DMA, votes in C, and logs timestamps |

## bench

The hardware-in-the-loop bench ([`EAGLE_Checklist-v1.md`](EAGLE_Checklist-v1.md), Act III): the Cyclone II board wired to 3 SPI IMUs (ICM-42688-P) and the barometer, an SPI link to the MCU, fault injection on one IMU, and oscilloscope / logic analyzer captures on GPIO toggle pins.

> ⚠️ Everything on the bench is 3.3 V.

## analysis

From logs to the paper's figures. From the plan, step 3: latency distribution histograms and CPU overhead plots in Python (`matplotlib`), covering the four metrics in §2 (jitter, WCET, MCU CPU load, fault recovery latency).

## paper

The benchmark paper (optional, see [`RESEARCH_PAPER_PLAN.md`](RESEARCH_PAPER_PLAN.md)), IEEE conference format. From the plan, step 4: abstract and introduction, system architecture, evaluation (histograms, WCET, fault-recovery waveforms), faculty review, submission to IEEE DASC / ESA PAC.
