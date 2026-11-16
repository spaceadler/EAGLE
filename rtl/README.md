# rtl

← [README](../README.md)

Synthesizable VHDL for the reflex core, in pure `ieee.numeric_std`. From [`RESEARCH_PAPER_PLAN.md`](../RESEARCH_PAPER_PLAN.md), step 1:

| File | Role |
|---|---|
| `spi_master.vhd` | SPI Mode 0 master with a configurable clock divider, one per IMU |
| `tmr_voter.vhd` | Single-cycle median vote across three 16-bit signed streams, threshold `G_MAX_DIFF` |
| `fault_detector.vhd` | 3-bit fault vector `o_fault_vec` for drifting or dead channels |
| `telemetry_tx.vhd` | Consensus data + fault status byte as an 8N1 serial stream |
| `eagle_reflex_top.vhd` | Top level: 3 SPI instances, the voter and the UART transmitter |

---
*powered by logic, coffee, and many sleepless nights*
