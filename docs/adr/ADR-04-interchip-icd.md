# ADR-04: Inter-chip links and interface control document

← [ADR index](../../README.md#decision-records)

| | |
|---|---|
| **Status** | Scoped |
| **Date** | 2026-11-22 |
| **Depends on** | [ADR-02](ADR-02-fault-tolerance.md) (pattern, broadcast and echo, out-of-band signals), [ADR-03](ADR-03-partitioning.md) (which tier samples, which MCU and its peripherals) |
| **Decides** | the link and protocol for each chip pair, who owns the sample clock, what a frame carries, the out-of-band pins and their levels while the FPGA configures, and the per-frame timing budget |

## Context

ADR-02 asks the links for four things: one sensor frame broadcast to every channel per control period;
every channel's reply echoing the sequence number and CRC of the input frame it used; a lockstep-error
signal and a heartbeat from each channel; reset and power control of each channel by the FPGA. This ADR
records the options and the arithmetic that constrains them. It fixes no bit layouts: the field list
below is what a frame has to carry, and the layout is the owner's.

## Questions

1. FPGA to MCU: SPI, UART or CAN FD, and which side is master?
2. Who owns time: does the FPGA trigger every frame and timestamp every sample?
3. What does each frame carry (fields, not layout)?
4. Which out-of-band pins, at which active level, and what must each read while the FPGA configures?
5. What is the per-frame timing budget at 500 Hz?
6. FPGA to sensors and actuators: SPI to the IMUs, DShot and PWM out.
7. MCU to Linux board, only if ADR-03 keeps Tier 1.

## Link options, FPGA to MCU

| Link | Timing and throughput | Integrity | Fit | Notes |
|---|---|---|---|---|
| SPI, FPGA master | synchronous on the FPGA's schedule; 64 B at 10 MHz is 51.2 µs | no CRC or addressing in the protocol: both go in the frame | every candidate MCU has SPI (the TMS570LC4357 has five MibSPI modules [1]); the MCU receives with DMA | the FPGA owns timing end to end |
| SPI, MCU master | the MCU pulls frames (the archived plan's choice) | as above | as above | time ownership moves to the MCU unless an FPGA "frame ready" pin starts each read |
| UART | asynchronous, 10 bits per byte with start and stop; 64 B at an assumed 3 Mbaud is 213 µs | parity per byte at most; CRC in the frame | every candidate; Ingenuity's Snapdragon talks to its MCUs over UART [2] | fewest wires; receiver clock tolerance limits the rate |
| Classical CAN, 1 Mbit/s | worst case (55 + 10·s) bit times per frame with s data bytes, so 135 µs for 8 bytes [3]; a 32-byte message is 4 frames, up to 540 µs | CRC and acknowledgement in hardware | TMS570 and RM57 have DCAN (CAN 2.0B) [1][4] | saturates at 500 Hz: see below |
| CAN FD | up to 64-byte payload, faster data phase [5] | hardware CRC | not on the TMS570 or RM57; on the S32K344 and some STM32 families (ADR-03) | excluded by ADR-02's P1 MCU class |
| SpaceWire (the archived plan) | | | no shortlisted MCU has a SpaceWire peripheral | dropped |

**Why classical CAN does not fit.** Take a 32-byte message each way (*assumption*). The downlink can be
one broadcast, because CAN is a bus: up to 540 µs. Two uplinks (P1) add up to 1,080 µs, three (P2) up to
1,620 µs. Since the downlink must arrive before the channels compute and the uplinks come after, P1 leaves
at most 2,000 − 1,620 = 380 µs of computation per 2 ms frame, and P2 leaves nothing.

## Time and sample ownership

- **Option under study: the FPGA is the time master.** A frame trigger every T_c starts the sensor reads,
  a free-running counter timestamps them, and one frame goes to every channel. ADR-02 gives the reason:
  channels that time themselves with ±50 ppm crystals drift a whole frame apart in about 20 s. Ingenuity's
  sixth flight gives the other: one lost image left every later image with a wrong timestamp, and the
  estimator followed them [6].
- **Shared IMU clock.** The ICM-42688-P accepts an external clock for its sample timing [7], so one FPGA
  pin can lock all three IMUs' output data rates to the FPGA's time base. The residual phase between the
  three IMUs' sampling instants still has to be measured (E4-3).
- **Counter widths.** A 32-bit timestamp at 50 MHz wraps after 2³² ÷ 50 MHz = 85.9 s, shorter than a
  flight; at 1 µs resolution it wraps after 71.6 minutes. A 16-bit sequence number wraps after 65,536
  frames, 131 s at 500 Hz; a 32-bit one after 99 days. Either the counters are wide enough for a flight,
  or every comparison uses serial-number arithmetic [8].

## Frame contents (fields, not layout)

**Downlink, FPGA to all channels, once per period**

| Field | Purpose |
|---|---|
| sync word | regain frame alignment after a slip |
| type and ICD version | refuse a mismatched build at boot |
| sequence number | monotonic frame counter; replies are matched against it |
| timestamp | FPGA time of the sample trigger |
| sensor data | 3 IMUs and the barometer, raw or after MVS (ADR-03 decides where MVS runs) |
| sensor status | per-IMU flags, MVS isolation result |
| supervisor state | mode, which channel is active, armed or not |
| CRC | over the whole frame |

**Uplink, each channel to the FPGA, once per period**

| Field | Purpose |
|---|---|
| sync word, type | as above |
| echo of the input sequence number and input CRC | input-congruence check (ADR-02, F6) |
| actuator commands | what the active channel wants the motors to do |
| channel status | self-test result, F´ health, rate-group slips |
| state copy (optional) | for rejoin after a power cycle (ADR-02 §5) |
| CRC | over the whole frame |

**CRC.** A CRC of degree r catches every burst error up to r bits; beyond that, the polynomial should be
chosen by its Hamming distance at the actual frame length, which differs a lot between common
polynomials [9]. CRC-8 against CRC-16 is decided on that table once the frame length is known.

## Out-of-band pins

| Signal | Direction | Active level | Must read during FPGA configuration | Purpose |
|---|---|---|---|---|
| HB | MCU to FPGA | toggle | (input to the FPGA) | heartbeat; F´'s Svc.Health strokes its watchdog output only while every pinged component answers [10] |
| nERROR | MCU to FPGA | low | (input to the FPGA) | the TMS570's error-signalling module pin [1] |
| ACTIVE | FPGA to MCU | to decide | "not active" | tells each channel whether its commands drive the motors |
| MCU_RST_N | FPGA to MCU | low | low: MCU held in reset | needs an external pull-down, see below |
| PWR_EN | FPGA to the channel's regulator | high | policy: off until the FPGA is up | power-cycles a channel |
| FPGA_OK | FPGA to MCU | to decide | "not OK" | tells the channels the supervisor is configured and passed its self-check |
| motor and pyro enables | FPGA to drivers | low (the workspace's active-low convention) | inactive | arming path |

**Levels while the FPGA configures.** Before and during configuration every EP2C5 user I/O is
tri-stated with a weak pull-up of 10 to 50 kΩ [11]. An active-low enable therefore reads inactive by
itself, which is the safe direction and fits the workspace's convention that a disconnected line reads
inactive. MCU_RST_N is the exception: pulled up, it would release the MCUs before the supervisor exists.
With the strongest pull-up (10 kΩ) and an external pull-down R_pd, the pin sits at
3.3 V · R_pd / (R_pd + 10 kΩ). To stay below an input-low threshold of 0.8 V (the LVTTL value, assumed
until the chosen MCU's datasheet is read):

```text
R_pd ≤ 10 kΩ · 0.8 / (3.3 − 0.8) = 3.2 kΩ;   R_pd = 2.2 kΩ gives 0.60 V
```

The FPGA then drives that pin high against 2.2 kΩ, 1.5 mA, below the smallest drive setting (4 mA) of
a Cyclone II 3.3-V LVTTL output [11].

## Timing budget per frame at 500 Hz

*Targets and derivations; the 64-byte frame is an assumption.*

| Step | Time | Origin |
|---|---|---|
| Frame trigger to the end of the IMU burst reads (3 in parallel, 15 bytes at 10 MHz) | 12 µs | derived |
| Frame build and CRC, one byte per clock at 50 MHz | 1.3 µs | derived |
| Broadcast, 64 B at 10 MHz SPI | 51 µs | derived |
| MCU computation | ≤ 1,000 µs | target; WCET measured in ADR-05 |
| Uplink, 64 B at 10 MHz | 51 µs | derived |
| Arrival window W | ≤ 500 µs | assumption from ADR-02 |
| Select or vote | < 1 µs | a few FPGA clocks |
| DShot600 frame to each ESC | 26.7 µs | [12] |
| **Total** | **about 1.64 ms of the 2 ms period** | |

This is the frame-path latency. The control loop also sees the half-sample hold of a 500 Hz loop
(1 ms) and the IMU's filter delay, the same items as in the Mars Helicopter budget (ADR-01 §1).

## Evidence that would settle this ADR

| Id | Evidence |
|---|---|
| E4-1 | GHDL simulation of the frame engine with corrupted, missing, late and reordered frames (after Phase A) |
| E4-2 | Logic-analyzer capture of the real link timing, including the MCU's DMA latency |
| E4-3 | Spread of the three ICM-42688-P sampling instants on a shared external clock |
| E4-4 | What the chosen ESC does when DShot frames stop (ADR-01) |
| E4-5 | nERROR latency on the chosen lockstep MCU, from TI's documentation or by fault injection |
| E4-6 | The chosen MCU's input thresholds, to fix R_pd |

## Sources

1. TI, TMS570LC4357 product page (five MibSPI, four DCAN, error-signalling module and nERROR pin).
   <https://www.ti.com/product/TMS570LC4357>
2. J. Balaram et al., "Mars Helicopter Technology Demonstrator," AIAA 2018, DOI 10.2514/6.2018-0023.
   <https://rotorcraft.arc.nasa.gov/Publications/files/Balaram_AIAA2018_0023.pdf>
3. R. I. Davis, A. Burns, R. J. Bril, J. J. Lukkien, "Controller Area Network (CAN) schedulability
   analysis: Refuted, revisited and revised," Real-Time Systems 35:239–272, 2007,
   DOI 10.1007/s11241-007-9012-7; worst-case frame time with 11-bit identifiers C = (55 + 10·s)·τ_bit,
   as stated in the authors' presentation <https://www-users.york.ac.uk/~rd17/papers/UpdatedCANPresentation.pdf>.
4. TI, RM57L843 product page. <https://www.ti.com/product/RM57L843>
5. CAN in Automation, "CAN FD: the basic idea." <https://can-cia.org/can-knowledge/can-fd-the-basic-idea>
6. H. F. Grip, "Surviving an In-Flight Anomaly: What Happened on Ingenuity's Sixth Flight," NASA, 2021.
   <https://science.nasa.gov/blog/surviving-an-in-flight-anomaly-what-happened-on-ingenuitys-sixth-flight/>
7. TDK InvenSense, ICM-42688-P product brief PB-000072. <https://www.farnell.com/datasheets/3045877.pdf>
8. R. Elz, R. Bush, "Serial Number Arithmetic," RFC 1982, 1996.
9. P. Koopman, T. Chakravarty, "Cyclic Redundancy Code (CRC) Polynomial Selection for Embedded
   Networks," DSN 2004, pp. 145–154, DOI 10.1109/DSN.2004.1311885.
10. F´ v4.3.0, `Svc/Health/docs/sdd.md`, requirement HTH-007.
11. Altera, Cyclone II Device Handbook, Volume 1, 2007, Table 2–16 (LVTTL drive strengths 4 to 24 mA),
    Table 5–3 (R_CONF) and Chapter 13 (weak pull-ups before and during configuration). <https://www.cs.columbia.edu/~sedwards/classes/2008/4840/cyc2_cii5v1.pdf>
12. "DShot and Bidirectional DShot," brushlesswhoop.com. <https://brushlesswhoop.com/dshot-and-bidirectional-dshot/>

## History

| Date | Change |
|---|---|
| 2026-11-22 | Scoped |
