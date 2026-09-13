# FALCON Project Checklist

## Act I: The RTOS Compute Core
*Establishing the foundational software environment on the microcontroller.*
- [ ] **Setup Toolchain:** Drop the Arduino IDE and configure a bare-metal ARM GCC toolchain (or PlatformIO/STM32Cube) for your MCU.
- [ ] **Port FreeRTOS:** Compile and run FreeRTOS on the MCU to enable preemptive multitasking.
- [ ] **F' Compilation:** Cross-compile NASA's F' framework so it runs natively on your FreeRTOS target.
- [ ] **F' Hello World:** Deploy a basic F' component that transmits a serial heartbeat down to your laptop to prove the stack works.

## Act II: The FPGA Reflex System & TMR
*Building the deterministic hardware abstraction layer in custom silicon.*
- [ ] **VHDL SPI Master:** Write an SPI master controller in VHDL to continuously poll the Barometer and the 3x IMUs.
- [ ] **TMR Voting Logic:** Implement SpaceX-style Triple Modular Redundancy in VHDL to compare the 3 IMUs simultaneously and instantly vote out any failing sensor.
- [ ] **VHDL PWM Generator:** Develop a PWM module in VHDL to accurately drive the airbrake servos and the system buzzer.
- [ ] **SpaceWire Node:** Implement the SpaceWire protocol (Data-Strobe) on the FPGA to pack sensor data and await read-requests.

## Act III: The Aerospace Bridge
*Integrating the hardware and software across physical buses.*
- [ ] **Physical Wiring:** Connect the SpaceWire Data and Strobe lines between the MCU and the FPGA.
- [ ] **F' SpaceWire Driver:** Write a C++ F' component to request and receive data from the FPGA over SpaceWire at a fixed rate (e.g., 100Hz).
- [ ] **Compliance Audit:** Run a static analyzer on the C++ driver to verify zero dynamic memory allocation, strict MISRA C++ adherence, and NASA Power of 10 compliance.

## Act IV: State Machine & Telemetry
*Building the flight logic and data logging systems.*
- [ ] **F' SD Card Driver:** Integrate or write an SPI driver in F' for continuous, high-speed flight data logging to the MicroSD card.
- [ ] **Rocket State Machine:** Implement the flight states (IDLE, ARMED, ASCENT, APOGEE, DESCENT, TOUCHDOWN) as a dedicated F' component.
- [ ] **Sub-GHz Radio Driver:** Wire a CC1101 radio to the MCU and write an F' driver to broadcast live CCSDS telemetry frames.
- [ ] **Flipper Zero Ground Station:** Configure your Flipper Zero's Sub-GHz radio to receive and decode the live telemetry payloads during flight.

## Act V: Guidance, Navigation, & Control
*Fusing sensor data and actuating control surfaces.*
- [ ] **Kalman Filter Component:** Write an F' math component to fuse noisy Barometer data and IMU drift into a highly accurate altitude/velocity estimate.
- [ ] **Airbrake PID Controller:** Implement a PID loop that calculates the error between your Target Altitude and Current Altitude to command the FPGA servo deflections.
- [ ] **Matlab/Python Simulation:** Feed simulated flight profiles (Software-in-the-Loop) into your F' components on your laptop to verify the math before testing physical hardware.

## Act VI: Advanced Aerospace Protocols
*Upgrading the internal networking to industry standards.*
- [ ] **CAN Bus Node:** Write a VHDL CAN controller for prioritized, reliable telemetry broadcasting across the system.
- [ ] **SpaceWire Router:** Expand the FPGA's SpaceWire node into a full router, allowing you to connect external payloads like cameras or secondary sensors exactly like a satellite bus.
