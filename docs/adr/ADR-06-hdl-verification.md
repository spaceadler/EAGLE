# ADR-06: HDL and software-first verification

← [ADR index](../../README.md#decision-records)

| | |
|---|---|
| **Status** | Scoped |
| **Date** | 2026-11-22 |
| **Depends on** | [ADR-03](ADR-03-partitioning.md) (the FPGA part and what logic it holds), [ADR-05](ADR-05-flight-software.md) (the SITL platform), [ADR-04](ADR-04-interchip-icd.md) (what the frame engine must do) |
| **Decides** | the HDL for the supervisor, the simulation and fault-injection setup, the formal checks, the software-in-the-loop setup and any hardware/software co-simulation |

## Context

ADR-02 makes the FPGA the trusted element and asks it to stay small enough to verify exhaustively. This
ADR decides how that claim gets its evidence. The order is software first: every block, every fault
path and the whole frame loop run in simulation before a board is wired.

The owner is learning VHDL and prefers it for its strong typing and its standing in European space and
avionics work, while knowing that the ETH systems groups' FPGA work is in SystemVerilog and HLS. The
table below checks both reasons.

## Questions

1. VHDL-2008 or SystemVerilog for the EP2C5, or for its replacement if ADR-03 changes the part?
2. Unit and fault-injection simulation: GHDL with VUnit, cocotb, or both?
3. Formal checks of the supervisor's state machine: which tools, which properties?
4. SITL: where does the plant model come from, and what drives simulated time?
5. Co-simulation of the FPGA model with the F´ SITL: worth it, and for which tests?

## Toolchain facts for the EP2C5

| Fact | Consequence |
|---|---|
| Quartus II 13.0sp1 is the last release that supports Cyclone II; support was removed in 13.1 [1] | one toolchain, frozen since 2013 |
| Quartus is an x86 program | synthesis runs on the x86 laptop or the Windows machine; the Raspberry Pi can only simulate |
| Earlier Quartus projects in this workspace found Quartus 13 reading VHDL-93 by default and rejecting non-ASCII characters even in comments | sources stay ASCII; VHDL-2008 needs a project setting, and Intel documents its VHDL-2008 support as a list of supported constructs [2] |
| SystemVerilog support in Quartus II 13.0 is also partial; the exact list was not checked | E6-1 |
| Yosys's `synth_intel` targets MAX 10, Cyclone 10 LP and Cyclone IV only [3]; nextpnr's only Intel target is Cyclone V, through Mistral, marked experimental [4] | there is no open path from HDL to an EP2C5 bitstream |
| GHDL 2.0.0 is installed on the Pi, and its `synth` command is present (checked locally) | open simulation now, and a netlist for formal checks through `ghdl-yosys-plugin`, whose authors call GHDL's synthesis beta and its PSL support partial [5] |

## HDL options

| | VHDL-2008 | SystemVerilog |
|---|---|---|
| Typing | strong: enumerated FSM states, records and ranged integers are checked by the compiler | weaker by default (implicit conversions, `logic` everywhere); closer with `typedef enum`, packages and `` `default_nettype none `` |
| EP2C5 toolchain | Quartus II 13.0sp1: VHDL-93 by default, a 2008 subset | Quartus II 13.0sp1: a subset |
| Open simulation | GHDL (installed on the Pi), NVC | Verilator, Icarus Verilog (neither installed; both cover SystemVerilog partially) |
| Open formal | GHDL with `ghdl-yosys-plugin` and SymbiYosys, PSL properties [5] | Yosys's own front end (a subset) with SymbiYosys |
| Avionics assurance | DO-254 is language-neutral: FAA Order 8110.105A asks the applicant to define coding standards for whichever HDL is used and to review conformance [6][7] | same |
| European space practice | ESA's VHDL Modelling Guidelines (ASIC/001, 1994, VHDL-93) [8]; Gaisler's GRLIB IP library and the LEON3 processor are VHDL [9] | |
| ETH systems research | | Coyote, "an operating system for FPGAs": hardware in SystemVerilog (`hw/hdl/*.sv`), user logic in RTL or HLS, built with AMD Vivado [10] |
| This workspace | every synthesizable repo is VHDL (`UART_TX`, `state-machine-counter`, `vHDL-mini-project`) | none |

The DO-254 reason does not hold as stated: the guidance does not favour VHDL. The European-space reason
does (ESA's guidelines, GRLIB). The ETH reason points to SystemVerilog, but on this part SystemVerilog
gets the same frozen, partial toolchain as VHDL; it would matter more after a move to a newer FPGA.

## Verification layers, software first

1. **Unit simulation** of each block, self-checking: GHDL with VUnit (MPL-2.0, GHDL among its supported
   simulators [11]), or cocotb in Python (GHDL 2.0 and later through VPI, which hides some VHDL-specific
   constructs such as 9-value signals [12]).
2. **Fault injection in simulation**: corrupted, missing, late and reordered frames; a stuck or late
   nERROR; heartbeat loss; IMU faults for MVS; the CRC_ERROR path. The configuration window (every pin at
   its resistor-defined level, ADR-04) is a board-level behaviour, checked against a model of the pins.
3. **Formal checks of the supervisor state machine** with SymbiYosys, bounded and unbounded. Candidate
   properties: at most one channel drives the actuators; the active channel changes only on a logged
   fault; a channel held in reset is never selected; after nERROR, the faulted channel is not selected
   again until it has passed the rejoin proof (ADR-02 §5). These proofs are the evidence for ADR-02's
   "exhaustively verifiable".
4. **SITL**: the F´ commanded path on Linux (a supported F´ platform) against a plant model of the
   ADR-01 vehicle, stepped at the 500 Hz frame rate by a simulated frame source, with faults injected at
   the frame level.
5. **Co-simulation (optional)**: the GHDL model of the FPGA, driven by cocotb, exchanging frames with the
   F´ SITL over a socket. It is slow, so it suits protocol and fault-path tests; long flights stay in
   the plain SITL.
6. **HIL**, after Phase A: the real FPGA and MCUs against the plant model on a PC (ADR-01's validation
   path).

## Evidence that would settle this ADR

| Id | Evidence |
|---|---|
| E6-1 | The Quartus II 13.0 handbook's VHDL-2008 and SystemVerilog support lists, then a small construct test synthesized for each (after Phase A) |
| E6-2 | Run times on the Pi of GHDL with VUnit and of cocotb for a frame-engine testbench |
| E6-3 | The formal flow (Yosys, `ghdl-yosys-plugin`, SymbiYosys, a solver) installed on the laptop and one property proved on a toy FSM |
| E6-4 | Whether the plant model and F´ SITL keep 500 Hz in real time on the Pi and on the laptop, or have to be stepped |
| E6-5 | ADR-03's decision on the part: a flash FPGA brings its own vendor toolchain, and the table above is redone |

## Sources

1. Intel, "Quartus Software Latest Version for Device Support," and knowledge base article 000073717.
   <https://www.intel.com/content/www/us/en/support/programmable/support-resources/programming/tls-software.html>
2. Intel, Quartus help, "Support for VHDL 2008" (the supported-construct list; later versions).
   <https://quartushelp.altera.com/current/hdl/vhdl/vhdl_list_2008_vhdl_support.htm>
3. YosysHQ, Yosys command reference, Intel techlibs (`synth_intel -family`).
   <https://yosyshq.readthedocs.io/projects/yosys/en/latest/cmd/index_techlibs_intel.html>
4. YosysHQ, nextpnr README. <https://github.com/YosysHQ/nextpnr>
5. T. Meissner, `psl_with_ghdl` (PSL with GHDL and SymbiYosys; status notes).
   <https://github.com/tmeissner/psl_with_ghdl>
6. Siemens Verification Horizons, "HDL Coding Standards for DO-254," 2021 (quotes FAA Order 8110.105A).
   <https://blogs.sw.siemens.com/verificationhorizons/2021/10/28/hdl-coding-standards-for-do-254/>
7. FAA, Advisory Circular 20-152A. <https://www.faa.gov/documentLibrary/media/Advisory_Circular/AC_20-152A.pdf>
8. P. Sinander, "VHDL Modelling Guidelines," ESA ASIC/001, Issue 1, September 1994.
9. Cobham Gaisler, GRLIB IP library and LEON3. <https://gaisler.com/grlib>
10. Systems Group, ETH Zurich, Coyote. <https://github.com/fpgasystems/Coyote>
11. VUnit. <https://vunit.github.io/> and <https://vunit.github.io/installing.html>
12. cocotb, "Simulator Support." <https://docs.cocotb.org/en/stable/simulator_support.html>

## History

| Date | Change |
|---|---|
| 2026-11-22 | Scoped |
