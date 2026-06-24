# ADS1293 Setup with STM32 H743ZI2

Setup and configuration guide for 3× ADS1293 CJMCU cardiac acquisition 
modules with STM32 Nucleo-144 H743ZI2 via SPI-DMA.

## 🔧 Hardware
- **Microcontroller:** STM32 Nucleo-144 H743ZI2 (ARM Cortex-M7, 480 MHz)
- **ADC Module:** CJMCU ADS1293 × 3
- **Communication:** SPI1 Full-Duplex Master + DMA
- **IDE:** STM32CubeIDE
- **Language:** Embedded C

## 📌 Pin Configuration
| No | STM32 Pin | Function |
|---|---|---|
| 1 | PA5 | SCLK |
| 2 | PA6 | SDO (MISO) |
| 3 | PB5 | SDI (MOSI) |
| 4 | PG3 | CSB ADS1293 (1) |
| 5 | PG0 | CSB ADS1293 (2) |
| 6 | PG1 | CSB ADS1293 (3) |
| 7 | PG2 | DRDYB (Data Ready) |

## ⚙️ SPI Configuration
- **Mode:** Full-Duplex Master
- **Frame Format:** Motorola
- **Data Size:** 8 Bits
- **First Bit:** MSB First
- **Prescaler:** 128 → Baud Rate 937.5 KBits/s
- **Clock Polarity (CPOL):** Low
- **Clock Phase (CPHA):** 1 Edge
- **Hardware NSS:** Disabled (manual CS via GPIO)

> Prescaler 128 dipilih untuk menyesuaikan kebutuhan sampling ECG 12-lead 
> pada 522 Hz, bukan kecepatan tinggi.

## 🕐 Clock Configuration
- **System Clock:** 480 MHz (HSE + PLL)
- **SPI1,2,3 Clock:** 120 MHz
- **Input Frequency:** 8 MHz (HSE)

## 🔌 GPIO Configuration
**CSB Pins (PG0, PG1, PG3):**
- Mode: Output Push Pull
- Output Level: High (default inactive)
- Pull: No pull-up / pull-down

**DRDYB Pin (PG2):**
- Mode: External Interrupt — Falling Edge
- Pull: Pull-up
- User Label: DRDYB

> DRDYB dikonfigurasi sebagai EXTI (interrupt) bukan polling. 
> ADS1293 akan men-trigger interrupt setiap data baru siap dibaca, 
> sehingga STM32 tidak perlu terus-menerus mengecek status — lebih 
> efisien dan responsif. DRDYB bersifat active low, sehingga falling 
> edge digunakan sebagai trigger.

## 📷 Screenshots
### SPI Parameter Settings
![SPI Config](docs/spi_config.png)

### GPIO Settings
![GPIO Config](docs/gpio_config.png)

### Clock Configuration
![Clock Config](docs/clock_config.png)

### Pin Assignment (CubeMX)
![Pin Assignment](docs/pin_assignment.png)

## 📁 Repository Structure
