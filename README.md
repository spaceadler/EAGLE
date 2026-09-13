<div align="center">

# FALCON Rocket Flight Controller

![Focus](https://img.shields.io/badge/Focus-Aerospace-purple)
![Language](https://img.shields.io/badge/C%2B%2B-00599C?style=flat-square&logo=c%2B%2B&logoColor=white)
![Language](https://img.shields.io/badge/VHDL-A50034?style=flat-square)
![Framework](https://img.shields.io/badge/F'_(F_Prime)-blue?style=flat-square)
![OS](https://img.shields.io/badge/FreeRTOS-lightgrey?style=flat-square)

</div>

---

## About
FALCON is a hybrid hardware-software flight controller designed for amateur rocketry, built from first principles to mimic real-world aerospace architectures (like those used by SpaceX and NASA). 

Instead of relying on Linux abstractions or 8-bit Arduino limits, FALCON separates the compute workload from the physical reflex system. It uses an RTOS-based microcontroller (The Brain) to handle NASA's F' framework, state management, and flight math, while a dedicated FPGA (The Reflex System) acts as a deterministic hardware abstraction layer to talk to physical sensors and servos at microsecond accuracy.

## Architecture Overview

```text
  [ THE COMPUTE CORE ]                      [ THE REFLEX SYSTEM ]
  ┌──────────────────┐    SpaceWire Link    ┌──────────────────┐
  │ Microcontroller  │ ◄──────────────────► │ FPGA (Cyclone II)│
  │ (FreeRTOS)       │   (Data-Strobe)      │                  │
  │                  │                      │ ┌──────────────┐ │
  │ - NASA F'        │                      │ │ TMR Voting   │ │
  │ - MISRA C++:2008 │                      │ └──────┬───────┘ │
  │ - Kalman Filter  │                      │        │         │
  │ - SD Card Log    │                      │  ┌─────┼─────┐   │
  └────────┬─────────┘                      │  ▼     ▼     ▼   │
           │                                │ IMU1  IMU2  IMU3 │
       [ SD Card ]                          │ (Baro)           │
                                            └─────────┬────────┘
                                                      │ PWM
                                                [ Airbrakes ]
```

## Features
- **NASA F' Framework on Bare Metal:** Cross-compiled to run natively on FreeRTOS.
- **SpaceX-Style TMR (Triple Modular Redundancy):** 3x IMUs polled in parallel by the FPGA, with VHDL voting logic to instantly discard failed sensors.
- **Hardware-Software Co-Design:** Uses the ESA SpaceWire protocol to bridge the microcontroller and the FPGA.
- **Aerospace Compliance:** All C++ driver and component code strictly adheres to **MISRA C++:2008** and **NASA's Power of 10** rules (no dynamic memory).
- **Live Telemetry:** Sub-GHz radio broadcasting down to a Flipper Zero Ground Station.

## Roadmap & Status
See [`FALCON_Checklist.md`](FALCON_Checklist.md) for the active 6-Act execution roadmap.

---

<div align="center">

*powered by logic, coffee, and many sleepless nights*

</div>
