# rtl

← [README](../README.md)

Synthesizable VHDL for the reflex core, in pure `ieee.numeric_std`. From [`EAGLE_Checklist.md`](../EAGLE_Checklist.md), Act I:

| File | Role |
|---|---|
| `spi_master.vhd` | SPI Mode 0 master with a configurable clock divider, one per IMU plus one for the barometer |
| `tmr_voter.vhd` | Single-cycle median vote across three 16-bit signed streams, threshold `G_MAX_DIFF` |
| `fault_detector.vhd` | 3-bit fault vector `o_fault_vec` for drifting or dead channels |
| MCU link | SPI slave: the MCU reads the voted data and the fault vector with DMA |
| PWM | Airbrake servos and buzzer |
| `eagle_reflex_top.vhd` | Top level: the SPI masters, the voter, the MCU link and the PWM |

---
*powered by logic, coffee, and many sleepless nights*
