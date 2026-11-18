# fw

← [README](../README.md)

MCU firmware: FreeRTOS + F´ ([`EAGLE_Checklist.md`](../EAGLE_Checklist.md), Acts II–III). For the optional benchmark, two builds that differ only in where the sensors are read and voted:

| Build | What it does |
|---|---|
| Co-design | Reads the FPGA's voted frames over SPI with DMA and logs timestamps |
| Baseline | Reads the 3 sensors itself over SPI with DMA, votes in C, and logs timestamps |

---
*powered by logic, coffee, and many sleepless nights*
