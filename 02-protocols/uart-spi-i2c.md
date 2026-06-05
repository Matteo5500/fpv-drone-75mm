# UART, SPI, I2C — Communication Protocols

## The Core Problem

Every peripheral on the drone (gyroscope, radio receiver, video transmitter)
needs to exchange data with the flight controller. These three protocols are
the communication buses that make this possible, each with different trade-offs.

---

## UART — Universal Asynchronous Receiver/Transmitter

**Type:** Asynchronous serial  
**Wires:** 2 (TX, RX) + GND  
**Topology:** Point-to-point only  
**Speed:** up to 921600 baud (921,600 bits/sec)

### How it works
Both chips agree on baud rate in advance. No shared clock.
Each byte is framed with start/stop bits:

```
IDLE  START  D0  D1  D2  D3  D4  D5  D6  D7  STOP
HIGH   LOW   [        8 data bits           ]  HIGH
```

### Usage on this drone
| UART | Peripheral | Protocol | Baud Rate |
|------|------------|----------|-----------|
| UART1 | ExpressLRS RX | CRSF | 420000 |
| UART2 | VTX SmartAudio | MSP/SA | 115200 |
| UART3 | Available | — | — |

### Betaflight source reference
File: `src/main/drivers/serial_uart.c`
Each UART is abstracted as a `uartPort_t` struct with circular TX/RX buffers.

---

## SPI — Serial Peripheral Interface

**Type:** Synchronous serial  
**Wires:** 4 (SCLK, MOSI, MISO, CS)  
**Topology:** 1 master, multiple slaves (1 CS pin per slave)  
**Speed:** up to 24MHz on STM32F411

### How it works
Master generates clock (SCLK). Data flows simultaneously in both
directions (MOSI and MISO) — full duplex. CS pin selects which
slave is active (LOW = active).

### Why SPI for the gyroscope
Gyro must be read at loop rate (1–8kHz = every 125–1000 µs).
SPI at 24MHz provides sufficient bandwidth. UART cannot.

### Usage on this drone
| SPI Bus | Peripheral | Speed |
|---------|------------|-------|
| SPI1 | ICM-42688-P (IMU) | 24MHz |
| SPI2 | Flash (Blackbox) | 21MHz |

### Betaflight source reference
File: `src/main/drivers/accgyro/accgyro_spi_icm426xx.c`
Function: `icm426xxGyroReadSPI()` — reads 6 bytes (X,Y,Z axes × 2 bytes each)

---

## I2C — Inter-Integrated Circuit

**Type:** Synchronous serial  
**Wires:** 2 (SDA, SCL)  
**Topology:** 1 master, up to 127 slaves (addressed)  
**Speed:** 100kHz (standard), 400kHz (fast mode)

### How it works
Every device has a 7-bit address. Master initiates all transfers.
Two wires shared by all devices on the bus (open-drain with pull-up resistors).

### Why I2C is NOT used for the gyroscope
Too slow for high-frequency sensor reading.
400kHz vs 24MHz SPI = 60x slower.

### Usage on this drone
| I2C Bus | Peripheral | Address |
|---------|------------|---------|
| I2C1 | Barometer (if present) | 0x76 |

---

## Protocol Selection Summary

| Requirement | Chosen Protocol | Reason |
|-------------|----------------|--------|
| High-frequency gyro read (1–8kHz) | SPI | Speed: up to 24MHz |
| Radio commands (CRSF, low latency) | UART | Simple, dedicated channel |
| VTX control (SmartAudio) | UART | Low bandwidth, simple |
| Barometer (1–10Hz) | I2C | Low speed acceptable, saves pins |

---

## Key Insight

UART ports are a **finite, exclusive resource**.  
The STM32F411 has 3 usable UARTs — each peripheral requiring
serial communication must be assigned a unique UART.  
Running out of UARTs means a peripheral cannot be used.
This constraint directly affects feature planning in Betaflight.