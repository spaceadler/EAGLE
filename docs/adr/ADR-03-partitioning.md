# ADR-03: Hardware/software partitioning and FPGA budget

← [ADR index](../../README.md#decision-records)

| | |
|---|---|
| **Status** | Scoped |
| **Date** | 2026-11-22 |
| **Depends on** | [ADR-01](ADR-01-mission-profile.md) (camera and tiers, actuators, IMU count), [ADR-02](ADR-02-fault-tolerance.md) (pattern, number of MCUs, lockstep or not, voter or switch) |
| **Decides** | how many tiers there are, which tier owns each function, what the FPGA's resource budget is, and which microcontroller and optional Linux board are used |

## Context

ADR-02 (Proposed) recommends pair-and-spare: two lockstep MCUs and an FPGA that selects, holds, watches
and power-cycles. That recommendation fixes the shape of this ADR but not its answers: if the owner
picks 2-out-of-3 voting instead, the FPGA gains an aligner and a voter and the MCU class opens up. This
ADR therefore records the numbers that hold for every option now and leaves the choices for when
ADR-01 and ADR-02 are Accepted.

## Questions

1. Two tiers (FPGA and MCUs) or three (plus a Linux board)? ADR-01 recommends no camera, which means two.
2. Which tier owns each function? Every function lives on exactly one tier.
3. What is the FPGA budget (logic elements, flip-flops, M4K blocks, multipliers, PLLs, I/O, clock), with
   every resource under 70 to 75% utilisation?
4. Is the EP2C5 the supervisor for free flight, or only for rig flights (ADR-01's R4)?
5. Which microcontroller? With ADR-02's P1 the choice is limited to lockstep parts.
6. If there is a Tier 1: which board, and what is it trusted with?

## Function map

"Follows" means an upstream ADR (still Proposed) decides it; "open" means this ADR decides it.

| Function | FPGA | MCU | Linux board | Status |
|---|---|---|---|---|
| Sensor sampling trigger, timestamps, one broadcast frame per period | candidate | possible (MCU as SPI master) | no | open: ADR-04 sample ownership and evidence E3-4 |
| Sensor MVS and plausibility checks | candidate | candidate | no | open: depends on ADR-01's IMU count and the LE budget |
| Watchdogs, lockstep-error monitor, channel select, MCU reset and power control | yes | no | no | follows ADR-02: the supervisor sits outside the channels it supervises |
| Hold-last and actuator frame generation (DShot, servo PWM) | candidate | possible | no | open: ADR-02's failover path favours the FPGA |
| Estimation, control laws, flight modes | no | yes | no | follows ADR-02's asymmetric simplicity; capacity alone would allow some of it (the EP2C5 has 13 multipliers) |
| Fault log | counters and a small RAM log | candidate (SD card or SPI flash) | candidate | open |
| Ground link (F´ GDS) | no | yes with two tiers | yes with three tiers | follows Q1 |
| Vision, autonomy | no | no | yes | only if ADR-01 brings in a camera |

## The FPGA: EP2C5T144C8

Confirmed from this workspace's Quartus projects (`state-machine-counter/top_level.qsf`:
`FAMILY "Cyclone II"`, `DEVICE EP2C5T144C8`, 3.3-V LVTTL I/O). All figures from the Cyclone II
Device Handbook [1] unless marked.

| Resource | EP2C5T144C8 | 75% ceiling | 70% ceiling |
|---|---|---|---|
| Logic elements (one 4-input LUT and one register each) | 4,608 | 3,456 | 3,226 |
| M4K RAM blocks (4 Kbit plus 512 parity bits each) | 26 (119,808 bits with parity, 106,496 without) | 19 | 18 |
| Embedded multipliers | 13 × (18 × 18), or 26 × (9 × 9) | 9 | 9 |
| PLLs | 2 | 1 | 1 |
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
board's header count (evidence E3-2).

### First-order block estimates

*Not synthesized; ±50%.* Basis: one LE holds a 4-input LUT and a register, and adders, counters and
comparators take about one LE per bit in arithmetic mode [1, ch. 2]. Fitter reports replace every row
(evidence E3-1).

| Block | Basis | LEs | M4K |
|---|---|---|---|
| SPI master, one IMU burst read | 16-bit shift register, clock divider, bit and byte counters, small FSM | 60 to 90 | 0 |
| Sensor frame store (3 IMUs, barometer) | dual-port RAM and addressing | 10 to 20 | 1 |
| Frame engine, per MCU link | shift register, 16-bit sequence counter, CRC-16 at 8 bits per clock, framing FSM | 120 to 180 | shared |
| Watchdog, per channel | 20-bit timeout counter (21 ms at 50 MHz), compare, latch | 30 to 40 | 0 |
| MVS, serial: one axis per clock over 6 axes | three 16-bit compares, a 16-bit 3:1 select, tolerance subtract and compare | 150 to 220 | 0 |
| MVS, fully parallel over 6 axes | six copies of the serial unit | 900 to 1,300 | 0 |
| A/B command compare, byte-serial | 8-bit compare, counter, mismatch latch | 20 to 40 | 0 |
| Hold-last and output select | last good command per channel in RAM, select | 40 to 80 | shared |
| DShot600 encoder, per motor | 16-bit shift register, bit timer, 4-bit CRC | 40 to 60 | 0 |
| Servo PWM, per channel, 1 µs steps | compare against a shared 15-bit counter | 20 to 30 | 0 |
| Supervisor FSM | modes, channel select, retry counters, reset and power control | 150 to 300 | 0 |
| TMR of supervisor state | triplicated registers and majority | 50 to 100 | 0 |
| Event log and 32-bit timestamp | 256 × 32-bit log, write pointer, counter | 50 to 80 | 2 |
| Debug UART transmitter (`UART_TX` repo) | as written | 40 to 60 | 0 |

Two envelopes, to show the budget is not the binding constraint (both with 3 IMUs, a barometer,
4 DShot channels, 2 servo channels and serial MVS):

| Pattern (ADR-02) | Main differences | LEs, mid estimate | Share of 4,608 | At +50% |
|---|---|---|---|---|
| P1, pair-and-spare | 2 MCU links, 2 watchdogs, select and compare | about 1,620 | 35% | about 2,430 (53%) |
| P2, 2-out-of-3 | 3 MCU links, 3 watchdogs, alignment, voter and resync (about 270) | about 2,050 | 44% | about 3,080 (67%) |

P2 with a fully parallel MVS would add about 900 LEs and pass the 75% ceiling at the high end of the
estimate. M4K use stays at 6 blocks or fewer, multipliers at zero (a Simplex envelope check of the form
xᵀPx < c for six states needs n² + n = 42 multiplications per frame, which one multiplier does in
under a microsecond at 50 MHz), PLLs at one.

### Is the EP2C5 a good fit?

| Criterion | Assessment |
|---|---|
| Capacity | fits both envelopes with room |
| Speed | C8 is the slowest grade; byte-serial logic at 50 MHz is a modest target, but timing closure still has to be shown (E3-1) |
| Configuration | weak fit for an always-on supervisor in the actuator path of an unstable vehicle: no upset immunity, about 160 ms cold start, 47 to 61 ms reconfiguration, weak pull-ups on every pin during configuration (ADR-02 §4) |
| Toolchain | one frozen, x86-only toolchain from 2013 with partial VHDL-2008 support (ADR-06) |
| Cost of keeping it | none: it is owned, and the tools are free |

Alternatives, for the owner to weigh if the configuration row decides it:

| Part family | Configuration | Relevance |
|---|---|---|
| Microchip ProASIC3, IGLOO2, PolarFire | flash or SONOS configuration cells, immune to neutron-induced configuration upsets, live at power-up [3][4] | the Ingenuity choice (ProASIC3L) |
| Intel MAX 10 | bitstream in on-chip flash, loaded into SRAM configuration at power-up ("instant-on"); CRC checking in user mode [5] | faster start, same upset exposure at run time |
| Lattice iCE40, ECP5 | SRAM, loaded from external flash | open toolchain (Yosys and nextpnr) [6]; same upset exposure |

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

## Optional Linux board (Tier 1)

Only if ADR-01 brings in a navigation camera. A Raspberry Pi running Linux is an F´ supported platform
with a reference project built in F´'s CI [7]. If it exists, it is trusted with nothing in the
commanded path: it sends set-points and receives telemetry, like Ingenuity's Snapdragon.

## Evidence that would settle this ADR

| Id | Evidence | Settles |
|---|---|---|
| E3-1 | Quartus II 13.0sp1 fitter reports (LEs, registers, M4K, Fmax at C8) for each block, once the RTL exists after Phase A | the budget table |
| E3-2 | Pin count from the EP2C5 board's schematic and headers, against ADR-04's pin list | I/O |
| E3-3 | The ADR-05 F´ port spike on each candidate MCU | the microcontroller |
| E3-4 | The benchmark from the archived paper plan ([`archive/RESEARCH_PAPER_PLAN.md`](../../archive/RESEARCH_PAPER_PLAN.md)): what changes in sample-time jitter, end-to-end latency and MCU load when triple-redundant sensor reading and voting move from MCU software to FPGA fabric? Its fairness rules carry over unchanged: same MCU, same build, same rate; the baseline reads the IMUs over SPI with DMA and votes in C; the only difference is where the sensors are read and voted. Metrics: sample-time jitter (GPIO at the sample instant, N = 100,000 cycles), end-to-end latency (histogram and observed maximum, which is not a proven WCET), MCU load (idle-task counter), fault-detection latency (injected corruption to flag). | whether sampling and voting belong in the FPGA |
| E3-5 | A measured CRC_ERROR pass time on the EP2C5 at the chosen check clock | how long F7 goes undetected |

## Sources

1. Altera, Cyclone II Device Handbook, Volume 1, February 2007: Tables 1–1 and 1–2 (resources,
   package I/O), ch. 2 (LE, arithmetic mode), ch. 3 (SEU detection), ch. 4 (POR), Table 5–3 (R_CONF),
   ch. 13 (configuration sizes and times).
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
| 2026-11-22 | Scoped |
