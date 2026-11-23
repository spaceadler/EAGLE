# ADR-03: Hardware/software partitioning and FPGA budget

← [ADR index](../../README.md#decision-records)

| | |
|---|---|
| **Status** | Scoped |
| **Date** | 2026-11-22 |
| **Depends on** | [ADR-01](ADR-01-mission-profile.md) (Accepted in part: sounding rocket, FlatSat, two tiers; sensors and pyro channels), [ADR-02](ADR-02-fault-tolerance.md) (pattern, fire chain, the three FPGA rules) |
| **Decides** | which tier owns each function, the FPGA's resource budget, whether the EP2C5 flies or only serves the FlatSat, and which microcontroller is used |

## Context

ADR-01 settled two tiers: an FPGA supervisor and RTOS microcontrollers, no Linux board, because there is
no camera. ADR-02 (Proposed) recommends pair-and-spare with lockstep MCUs and puts the pyro fire chain's
interlock, electronic arm and high-side switch in the FPGA and the low-side switch in the MCUs. This ADR
records the numbers that hold for every remaining option and leaves the choices for when ADR-02 is
Accepted.

## Questions

1. Which tier owns each function? Every function lives on exactly one tier.
2. What is the FPGA budget (logic elements, registers, M4K blocks, multipliers, PLLs, clocks, I/O), with
   every resource under 70 to 75% utilisation?
3. Does the EP2C5 fly, under ADR-02's three rules, or does it only serve the FlatSat, with a flash FPGA
   for flight?
4. Which microcontroller? Lockstep parts for P1; the field opens for P4.
5. Where does MVS run, and does the FPGA's interlock read raw barometer counts (coarse but independent
   of the MCUs) or the MCUs' compensated pressure (accurate but not independent)?
6. How does the FlatSat feed the sensors: emulators on the flight SPI buses, or an injection mode in the
   FPGA? (ADR-06 decides; this ADR carries the I/O cost.)

## Function map

"Follows" means an upstream ADR decides it (ADR-02 is still Proposed); "open" means this ADR decides it.

| Function | FPGA | MCU | Status |
|---|---|---|---|
| Sensor sampling trigger, timestamps, one broadcast frame per period | candidate | possible (MCU as SPI master) | open: ADR-04 sample ownership, evidence E3-4 |
| Sensor MVS and plausibility checks | candidate | candidate | open (Q5) |
| Watchdogs, lockstep-error monitor, channel select, MCU power control | yes | no | follows ADR-02: the supervisor sits outside the channels it supervises |
| Estimation, flight-phase machine, the decision *when* to fire | no | yes | follows ADR-02's asymmetric simplicity |
| Fire interlock (the decision *whether*), electronic arm, high-side drive, fire-pulse timing | yes | no | follows ADR-02 §5 |
| Low-side drive | no | yes, either MCU (wired-OR) | follows ADR-02 §5 (its owner decision 4) |
| Continuity and switch-state monitoring | digital comparator inputs | ADC readings | open |
| Airbrake servo PWM, if fitted | candidate | possible | open |
| Fault log | counters and a small RAM log | SD card or SPI flash | open |
| GNSS, if fitted | no | yes (UART) | open |
| Ground link (F´ GDS) | no | yes | follows ADR-01 (two tiers) |
| Backup deployment | – | – | an independent altimeter, outside EAGLE (ADR-01) |

## The FPGA: EP2C5T144C8

Confirmed from this workspace's Quartus projects (`state-machine-counter/top_level.qsf`:
`FAMILY "Cyclone II"`, `DEVICE EP2C5T144C8`, 3.3-V LVTTL I/O). All figures from the Cyclone II
Device Handbook [1] unless marked.

| Resource | EP2C5T144C8 | 75% ceiling | 70% ceiling |
|---|---|---|---|
| Logic elements (one 4-input LUT and one register each) | 4,608 | 3,456 | 3,226 |
| Registers in the logic array | 4,608, one per LE | 3,456 | 3,226 |
| M4K RAM blocks (4 Kbit plus 512 parity bits each) | 26 (119,808 bits with parity, 106,496 without) | 19 | 18 |
| Embedded multipliers | 13 × (18 × 18), or 26 × (9 × 9) | 9 | 9 |
| PLLs | 2 | 1 | 1 |
| Global clock networks | 8 on the EP2C5 (16 on larger Cyclone II parts) | 6 | 5 |
| User I/O, 144-pin TQFP | 89 | 66 | 62 |
| Speed grade | C8, the slowest in the family | | |
| Core supply | 1.2 V (1.15 to 1.25 V) | | |
| Board clock | 50 MHz oscillator (workspace board) | | |
| Configuration | SRAM, loaded at power-up; Active Serial from an EPCS device | | |
| Uncompressed bitstream | 1,265,792 bits (Table 13–3, marked preliminary); the AS timing example uses 1,223,980 bits | | |
| Cold start | power-on reset ≥ 100 ms (non-"A" part) + AS configuration ≤ 61.2 ms (47.1 ms typical) | | |
| Upset detection | CRC_ERROR pin: 32-bit CRC over configuration SRAM in user mode, check clock 400 kHz to 80 MHz | | |
| I/O before and during configuration | tri-stated with weak pull-ups, R_CONF 10 / 25 / 50 kΩ (min / typ / max) | | |
| Toolchain | Quartus II 13.0sp1, the last release with Cyclone II [2]; x86 only, so not on the Raspberry Pi | | |

The I/O row is the package limit. The cheap EP2C5T144 boards bring out fewer pins on their headers, and
some are taken by the oscillator, LEDs, buttons and the configuration device, so the real limit is the
board's header count (evidence E3-2). ADR-04's signal count comes to 51 to 66 for P1 and 61 to 79 for P2,
depending mostly on whether each sensor gets its own SPI bus: P1 reaches the 75% ceiling at its high end.

### First-order block estimates

*Not synthesized; ±50%.* Basis: one LE holds a 4-input LUT and a register, and adders, counters and
comparators take about one LE per bit in arithmetic mode [1, ch. 2]. Fitter reports replace every row
(evidence E3-1).

| Block | Basis | LEs | M4K |
|---|---|---|---|
| SPI master, one sensor burst read | 16-bit shift register, clock divider, bit and byte counters, small FSM | 60 to 90 | 0 |
| Sensor frame store (IMUs, high-g, barometer) | dual-port RAM and addressing | 10 to 20 | 1 |
| Frame engine, per MCU link | shift register, 16-bit sequence counter, CRC-16 at 8 bits per clock, framing FSM | 120 to 180 | shared |
| Watchdog, per channel | 20-bit timeout counter (21 ms at 50 MHz), compare, latch | 30 to 40 | 0 |
| MVS, serial: one axis per clock over 6 axes | three 16-bit compares, a 16-bit 3:1 select, tolerance subtract and compare | 150 to 220 | 0 |
| MVS, fully parallel over 6 axes | six copies of the serial unit | 900 to 1,300 | 0 |
| A/B output compare, byte-serial | 8-bit compare, counter, mismatch latch | 20 to 40 | 0 |
| Hold-last and output select (airbrake command) | last good command per channel, select | 20 to 40 | 0 |
| Fire interlock and electronic arm | liftoff threshold and persistence counter, minimum-time counter, no-thrust compare, pressure-rising counter, main-pressure compare, small FSM | 150 to 220 | 0 |
| Pyro channel, per channel | fire-pulse timer, high-side drive, state latch, monitor inputs | 30 to 50 | 0 |
| Servo PWM, per channel, 1 µs steps | compare against a shared 15-bit counter | 20 to 30 | 0 |
| Supervisor FSM | modes, channel select, retry counters, power control | 150 to 300 | 0 |
| TMR of supervisor and interlock state | triplicated registers and majority | 75 to 125 | 0 |
| Event log and 32-bit timestamp | 256 × 32-bit log, write pointer, counter | 50 to 80 | 2 |
| Debug UART transmitter (`UART_TX` repo) | the existing module, never synthesized for a count | 40 to 60 | 0 |

Envelopes, with 3 IMUs, one high-g accelerometer, one barometer, 2 pyro channels, one airbrake servo and
serial MVS:

| Pattern (ADR-02) | Main differences | LEs, mid estimate | Share of 4,608 | At +50% |
|---|---|---|---|---|
| P1, pair-and-spare | 2 MCU links, 2 watchdogs, select and compare | about 1,740 | 38% | about 2,610 (57%) |
| P4, asymmetric pair | as P1 without the lockstep-error inputs | about 1,740 | 38% | about 2,610 (57%) |
| P2, 2-out-of-3 | 3 MCU links, 3 watchdogs, alignment, voter and resync (about 270) | about 2,160 | 47% | about 3,240 (70%) |

A second high-g accelerometer and a second barometer add about 150 LEs. P2 with a fully parallel MVS would
pass the 75% ceiling at the high end of the estimate. M4K use stays at 6 blocks or fewer, multipliers at
zero, PLLs at one. The logic budget does not bind; the I/O count can, and the configuration behaviour does.
A classical CAN link (ADR-04) would add a CAN controller per link, not in the table above.

### Does the EP2C5 fly?

| Criterion | Assessment |
|---|---|
| Capacity | fits every envelope with room |
| Speed | C8 is the slowest grade; byte-serial logic at 50 MHz is a modest target, but timing closure still has to be shown (E3-1) |
| Configuration | acceptable for the rocket under ADR-02's three rules (barometer-confirmed apogee, MCU resets decoupled from FPGA configuration, a fast CRC check with interlock restore): 47 to 61 ms of reconfiguration fits every phase budget except the 8 ms boost estimate. Still a weak part for an always-on supervisor: no upset immunity, about 160 ms cold start, weak pull-ups on every pin while configuring |
| Toolchain | one frozen, x86-only toolchain from 2013 with partial VHDL-2008 support (ADR-06) |
| Cost of keeping it | none: it is owned, and the tools are free |

If the owner prefers upset immunity for flight, these families are the alternatives; the EP2C5 can stay
the FlatSat development part either way:

| Part family | Configuration | Relevance |
|---|---|---|
| Microchip ProASIC3, PolarFire | flash (ProASIC3) or SONOS (PolarFire) configuration cells, immune to neutron-induced configuration upsets [3][4]; no external configuration device; ProASIC3 is live at power-up [3] | the Ingenuity choice (ProASIC3L) |
| Intel MAX 10 | bitstream in on-chip flash, loaded into SRAM configuration at power-up ("instant-on"); CRC checking in user mode [5] | faster start, same upset exposure at run time |
| Lattice iCE40, ECP5 | SRAM, usually loaded from external SPI flash | open toolchain (Yosys and nextpnr) [6]; same upset exposure |

## Microcontroller shortlist

| Part (board) | Cores | Lockstep | Memory | CAN | F´ platform status [7][8] | RTOS paths found |
|---|---|---|---|---|---|---|
| TI TMS570LC4357 (LAUNCHXL2-570LC43) | 2 × Cortex-R5F, 300 MHz | yes | 4 MB flash, 512 KB RAM, both ECC; big-endian (BE32) | 4 × DCAN, CAN 2.0B, no CAN FD | no reference | FreeRTOS through TI HALCoGen (FreeRTOS 9.0.0 according to TI E2E user reports [11]); RTEMS 6 BSP `tms570lc4357_hdk` [12]; no Zephyr board found |
| TI RM57L843 (LAUNCHXL2-RM57L) | 2 × Cortex-R5F, 330 MHz | yes | 4 MB flash, 512 KB RAM, both ECC; little-endian | 4 × DCAN, CAN 2.0B | no reference | FreeRTOS reported (MathWorks support package for the RM57Lx LaunchPad); no Zephyr board found |
| NXP S32K344 | Cortex-M7, 160 MHz | yes, in lockstep mode (ASIL D; ASIL B when the cores run separately) [13] | 4 MB flash | CAN FD | no reference | not checked |
| STM32 (family open) | Cortex-M | none found; IEC 61508 through ST's X-CUBE-STL software self-test, SIL 2 in one channel, SIL 3 needs redundancy [14] | per family | FDCAN on some families | "STM32 baremetal: in development" (JPL, February 2026 [8]) | FreeRTOS, Zephyr |
| Raspberry Pi RP2040 | 2 × Cortex-M0+, 133 MHz, no FPU | no | 264 KB SRAM | none | Zephyr reference (Pi Pico) | Zephyr, FreeRTOS |
| Raspberry Pi RP2350 | 2 × Cortex-M33 with FPU, or 2 × Hazard3 RISC-V, 150 MHz | no | 520 KB SRAM | none | Zephyr reference (Pi Pico 2, on its RISC-V cores), built in F´ CI | Zephyr, FreeRTOS |

The TMS570 and RM57 rows come from TI's product pages [9][10]. TI also sells a space-enhanced plastic
TMS570LC4357-SEP. With ADR-02's P1, only the first three rows qualify, and none of them has an F´ port:
whichever one gets a working F´ deployment first in the ADR-05 port spike has a real claim to the slot.
If P4 takes over, the RP2350 and the Teensy 4.1 (also an F´ Zephyr reference built in CI [7]) come back in.

## The FlatSat's share of the partition

The FlatSat (ADR-01 §8, ADR-06) adds test equipment around the flight boards: e-match simulators on the
pyro channels, a current-logging supply, an HIL host, and, if ADR-06 picks emulation, sensor emulators that
answer on the flight SPI buses. Emulators sit on the same buses as the real sensors, so the flight FPGA
logic is the same on the bench and in flight; an injection mode inside the FPGA would add logic and pins
that must be impossible to enable in flight (ADR-06).

## Evidence that would settle this ADR

| Id | Evidence | Settles |
|---|---|---|
| E3-1 | Quartus II 13.0sp1 fitter reports (LEs, registers, M4K, Fmax at C8) for each block, once the RTL exists after Phase A | the budget table |
| E3-2 | Pin count from the EP2C5 board's schematic and headers, against ADR-04's pin list | I/O |
| E3-3 | The ADR-05 F´ port spike on each candidate MCU | the microcontroller, and P1 against P4 |
| E3-4 | The benchmark from the archived paper plan ([`archive/RESEARCH_PAPER_PLAN.md`](../../archive/RESEARCH_PAPER_PLAN.md)): what changes in sample-time jitter, end-to-end latency and MCU load when triple-redundant sensor reading and voting move from MCU software to FPGA fabric? Fairness rules unchanged: same MCU, same build, same rate; the baseline reads the IMUs over SPI with DMA and votes in C; the only difference is where the sensors are read and voted. Metrics: sample-time jitter (GPIO at the sample instant, N = 100,000 cycles), end-to-end latency (histogram and observed maximum, which is not a proven WCET), MCU load (idle-task counter), fault-detection latency (injected corruption to flag). Run on the FlatSat. | whether sampling and voting belong in the FPGA |
| E3-5 | A measured CRC_ERROR pass time on the EP2C5 at the chosen check clock | whether R4 holds through an FPGA upset at apogee (ADR-02 §4) |

## Sources

1. Altera, Cyclone II Device Handbook, Volume 1, February 2007: Tables 1–1 and 1–2 (resources,
   package I/O), ch. 2 (LE, arithmetic mode), ch. 3 (SEU detection), ch. 4 (POR), Table 5–3 (R_CONF),
   ch. 7 (eight global clocks on the EP2C5), ch. 13 (configuration sizes and times).
   <https://www.cs.columbia.edu/~sedwards/classes/2008/4840/cyc2_cii5v1.pdf>
2. Intel, "Quartus Software Latest Version for Device Support" (Cyclone II: 13.0sp1), and knowledge
   base article 000073717 (families removed from 13.1).
   <https://www.intel.com/content/www/us/en/support/programmable/support-resources/programming/tls-software.html>
3. Actel, "ProASIC3 Flash Family FPGAs," datasheet Advanced v0.6.
   <https://home.engineering.iastate.edu/~zambreno/classes/cpre583/2006/documents/actel/PA3_DS.pdf>
4. Microchip, AC478 "Microchip PolarFire FPGAs for Safety Critical."
   <https://www.microchip.com/content/dam/mchp/documents/FPGA/ApplicationNotes/ApplicationNotes/Microchip_PolarFire_FPGAs_for_Safety_Critical__AC478_Application_Note.pdf>
5. Intel, MAX 10 FPGA Configuration User Guide (internal configuration from CFM into CRAM, SEU
   mitigation). <https://www.intel.com/content/www/us/en/docs/programmable/683865/current/configuration.html>
6. YosysHQ, nextpnr README (iCE40 via Project IceStorm, ECP5 via Project Trellis; Cyclone V via
   Mistral, experimental). <https://github.com/YosysHQ/nextpnr>
7. F´ documentation, "Supported Platforms."
   <https://fprime.jpl.nasa.gov/latest/docs/user-manual/framework/supported-platforms/>
8. K. F. Ortega, "F Prime Platforms," JPL Advanced F´ Workshop, February 2026 (supported and targeted
   platforms; historical use of baremetal F´ on Ingenuity).
9. TI, TMS570LC4357 product page <https://www.ti.com/product/TMS570LC4357>, and the TMS570LC4357-SEP
   datasheet, SPNS254 <https://www.ti.com/lit/pdf/SPNS254>.
10. TI, RM57L843 product page. <https://www.ti.com/product/RM57L843>
11. TI E2E forum, "TMS570LS3137: FreeRTOS porting to TMS570LS3137" and related threads (HALCoGen
    generates a FreeRTOS 9.0.0 project; MPU port issues). User reports, not TI documentation.
    <https://e2e.ti.com/support/microcontrollers/arm-based-microcontrollers-group/arm-based-microcontrollers/f/arm-based-microcontrollers-forum/1601561/tms570ls3137-freertos-porting-to-tms570ls3137>
12. RTEMS 6.2 User Manual, BSPs, "arm/tms570." <https://docs.rtems.org/docs/6.2/user/bsps/arm/tms570.html>
13. Arm Keil device database, NXP S32K344; NXP SafeAssure community thread on S32K344 lockstep and
    ASIL metrics. <https://www.keil.arm.com/devices/nxp-s32k344/>
14. STMicroelectronics, X-CUBE-STL. <https://www.st.com/en/embedded-software/x-cube-stl.html>

## History

| Date | Change |
|---|---|
| 2026-11-22 | Scoped for the multirotor profile |
| 2026-11-22 | Rescoped for the sounding rocket and FlatSat: two tiers settled, ESC outputs replaced by the fire chain, envelopes recomputed, the EP2C5 reassessed under ADR-02's rules |
| 2026-11-23 | I/O count taken from ADR-04's rescoped signal count (51 to 66 for P1) |
