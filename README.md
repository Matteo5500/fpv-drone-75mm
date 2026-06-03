# 🚁 FPV Micro Drone 75mm — Engineering Documentation

> **Phase 1 — Software & Research** | Phase 2 — Hardware Assembly *(planned)*

Full engineering documentation for a 75mm 1S Tiny Whoop FPV drone, covering 
hardware architecture design, embedded firmware analysis, communication protocol 
deep-dives, and complete Betaflight CLI configuration scripting.

This project is structured as a two-phase build: Phase 1 is a zero-budget 
software-only research phase producing ready-to-flash configuration scripts and 
deep technical documentation. Phase 2 will involve physical component procurement, 
soldering, and assembly.

---

## 🎯 Project Goals

- Design a complete hardware architecture for a 75mm 1S Tiny Whoop from scratch
- Analyze and document embedded communication protocols (UART, SPI, I2C, DSHOT, ELRS/CRSF)
- Study Betaflight open-source firmware (C) — PID controller, gyroscope filters
- Produce a complete, commented Betaflight CLI configuration script ready for flashing
- Structure all findings as professional engineering documentation

---

## 🏗️ Repository Structure
fpv-drone-75mm/
│
├── 01-hardware/          # Component selection, BOM, compatibility analysis
├── 02-protocols/         # UART, SPI, I2C, DSHOT, ELRS/CRSF deep-dives
├── 03-pin-mapping/       # STM32F411 pinout and Betaflight resource mapping
├── 04-firmware-analysis/ # Betaflight C source code analysis (PID, filters)
├── 05-cli-configuration/ # Complete Betaflight CLI scripts, commented
└── 06-docs/              # Glossary, architecture diagrams, references

---

## 📦 Target Hardware Architecture

| Component | Category | Protocol |
|-----------|----------|----------|
| BetaFPV Meteor75 Pro FC (STM32F411) | Flight Controller AIO | — |
| 0802 brushless motors (4x) | Motors | DSHOT300 |
| ExpressLRS 2.4GHz RX (onboard) | Radio Receiver | CRSF/UART |
| Analog VTX 5.8GHz (onboard) | Video Transmitter | UART |
| 1S LiPo 300mAh | Power | — |

> ⚠️ Phase 1 is software-only. No hardware has been purchased yet.
> All configuration scripts are prepared theoretically and will be 
> validated during Phase 2 assembly.

---

## 🔬 Firmware

**Betaflight 4.5** — open source flight controller firmware written in C.
- Repository: [betaflight/betaflight](https://github.com/betaflight/betaflight)
- Target: `BETAFPVF411`
- Key modules studied: PID controller, gyroscope filtering, DSHOT driver

---

## 🗺️ Roadmap

### ✅ Phase 1 — Software & Research *(in progress)*
- [x] Repository structure and documentation setup
- [ ] Hardware component selection and compatibility analysis
- [ ] Communication protocols documentation
- [ ] STM32F411 pin mapping and resource allocation
- [ ] Betaflight firmware analysis (PID + gyro filters)
- [ ] Complete CLI configuration script

### 🔲 Phase 2 — Hardware Assembly *(planned — pending budget)*
- [ ] Component procurement
- [ ] Soldering and assembly
- [ ] Firmware flash and CLI script upload
- [ ] Test flights and PID tuning

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Firmware | Betaflight 4.5 (C) |
| MCU | STM32F411CEU6 |
| Radio Protocol | ExpressLRS 2.4GHz / CRSF |
| Motor Protocol | DSHOT300 |
| Configuration | Betaflight CLI |
| Documentation | Markdown |

---

## 📚 Key References

- [Betaflight Official Docs](https://betaflight.com/docs/wiki)
- [Betaflight GitHub Source](https://github.com/betaflight/betaflight)
- [ExpressLRS Documentation](https://www.expresslrs.org)
- [STM32F411 Reference Manual](https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)

---

## 👤 Author

**Matteo** — Junior Software Developer  
Building embedded systems knowledge from the ground up.  
[GitHub](https://github.com/Matteo5500)