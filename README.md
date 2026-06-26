# ADS1293 Setup with STM32 H743ZI2
### *3-Module Configuration for 12-Lead ECG Acquisition*

Setup and configuration of 3× CJMCU ADS1293 modules with STM32 Nucleo-144 H743ZI2 using SPI-DMA communication for simultaneous 12-lead ECG signal acquisition.

---

## 📸 Hardware Demo

### Wiring Connection
![Wiring](docs/wiring.jpg)

### STM32CubeIDE Configuration
| SPI Settings | GPIO Settings |
|---|---|
| ![SPI Config](docs/spi_config.png) | ![GPIO Config](docs/gpio_config.png) |

| Clock Configuration | Pin Assignment (CubeMX) |
|---|---|
| ![Clock Config](docs/clock_config.png) | ![Pin Assignment](docs/pin_assignment.png) |

---

## Features & Problems Solved

This project addresses complex technical challenges in embedded 12-lead ECG acquisition:

- **Multi-module SPI management:** Simultaneously managing 3 ADS1293 modules via SPI-DMA with 3 separate chip selects — zero data collision
- **Real-time acquisition:** Reading all 12 ECG leads simultaneously at 522 Hz sampling frequency using Embedded C
- **DRDYB interrupt handling:** Using EXTI interrupt instead of polling — STM32 is efficient and never wastes CPU cycles waiting for data
- **Register-level configuration:** Directly configuring ADS1293 sampling frequency via SPI register writes to match system requirements

---

## Why 3 ADS1293 Modules?

This was the most critical design decision in the project.

**1 ADS1293 module = 3 ECG channels**

To acquire **12-lead ECG**, the following mapping is used:

| Module | Role | Active Channels | Leads Produced |
|---|---|---|---|
| ADS1293 #1 | Master | 2 channels | Lead I, Lead II → Lead III, aVR, aVL, aVF (calculated mathematically) |
| ADS1293 #2 | Slave | 3 channels | V1, V2, V3 (precordial) |
| ADS1293 #3 | Slave | 3 channels | V4, V5, V6 (precordial) |

**Why are limb leads (I, II, III, aVR, aVL, aVF) covered by just 1 module?**

Because Lead I and Lead II alone are sufficient — the rest are derived:
- Lead III = Lead II − Lead I
- aVR = −(Lead I + Lead II) / 2
- aVL = Lead I − Lead II / 2
- aVF = Lead II − Lead I / 2

**Why do precordial leads (V1–V6) need 2 additional modules?**

V1–V6 are 6 independent leads that must be measured directly — they cannot be derived from other leads. Each module handles 3 leads, so 2 modules are required for V1–V6.

> If only 3-lead or 6-lead ECG is needed, a single ADS1293 module is sufficient.

---

## Hardware & Technology

| Component | Detail |
|---|---|
| **Microcontroller** | STM32 Nucleo-144 H743ZI2 (ARM Cortex-M7, 480 MHz) |
| **ADC Module** | CJMCU ADS1293 × 3 |
| **Communication** | SPI1 Full-Duplex Master + DMA |
| **IDE** | STM32CubeIDE |
| **Language** | Embedded C |
| **Filter Design** | MATLAB (for filter coefficient calculation) |

---

## Pin Configuration

| No | STM32 Pin | Function |
|---|---|---|
| 1 | PA5 | SCLK |
| 2 | PA6 | SDO (MISO) |
| 3 | PB5 | SDI (MOSI) |
| 4 | PG3 | CSB ADS1293 #1 (Master) |
| 5 | PG0 | CSB ADS1293 #2 (Slave) |
| 6 | PG1 | CSB ADS1293 #3 (Slave) |
| 7 | PG2 | DRDYB (Interrupt) |

---

## SPI Configuration

> **Note:** STM32CubeIDE must be configured manually. The settings below are the exact parameters used in this project.

| Parameter | Value | Reason |
|---|---|---|
| Mode | Full-Duplex Master | — |
| Prescaler | 128 | Matched to ECG sampling needs, not high-speed transfer |
| Baud Rate | 937.5 KBits/s | Sufficient for 3-module data transfer @ 522 Hz |
| CPOL | Low | Per ADS1293 datasheet specification |
| CPHA | 1 Edge | Per ADS1293 datasheet specification |
| Data Size | 8 Bits | ADS1293 register format |
| NSS | Disabled | CS managed manually via GPIO (3 separate CS pins) |

> Prescaler 128 was chosen to match the 522 Hz ECG sampling requirement — high baud rate is not needed here.

---

## GPIO Configuration

> **Note:** All GPIO settings below must be configured manually in STM32CubeIDE.

**CSB Pins (PG0, PG1, PG3) — Output Push Pull:**

Each ADS1293 module has its own dedicated CS pin so they can be accessed one at a time without interfering with each other. Default state is HIGH (inactive).

**DRDYB Pin (PG2) — External Interrupt, Falling Edge:**

DRDYB is configured as an interrupt, not polling. How it works:
1. STM32 operates normally while waiting
2. When ADS1293 has new data, DRDYB pin pulls LOW (HIGH → LOW)
3. This falling edge triggers the EXTI interrupt
4. STM32 briefly pauses, reads the data, then continues

This is far more efficient than constantly polling *"is data ready?"* every cycle. DRDYB is active-low per ADS1293 datasheet.

---

## Clock Configuration

> **Note:** Clock configuration must be set manually in STM32CubeIDE **Clock Configuration** tab. Refer to the screenshot above for the full clock tree.

| Parameter | Value |
|---|---|
| HSE Input | 8 MHz |
| System Clock (SYSCLK) | 240 MHz |
| CPU / AHB Clock (D1CPRE /2) | 120 MHz |
| SPI1, SPI2, SPI3 Clock | 120 MHz (sourced from PLL1Q) |

> SPI1 runs off the 120 MHz PLL1Q output. With Prescaler 128, the effective baud rate is **120 MHz / 128 = 937.5 KBits/s** — matching the SPI configuration above.

---

## ADS1293 Register Configuration

All ADS1293 parameters are configured via SPI register writes after power-on. No external tools needed — the STM32 writes directly to each module's registers over SPI.

### Master Module (ADS1293 #1 — Lead I & Lead II)

```c
void setup_ECG_Master(void) {
    // --- Input Channel Routing ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x01, 0x11);
    // FLEX_CH1_CN: CH1+ = IN2 (LA), CH1- = IN1 (RA) → measures Lead I = LA - RA

    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x02, 0x19);
    // FLEX_CH2_CN: CH2+ = IN3 (LL), CH2- = IN1 (RA) → measures Lead II = LL - RA

    // --- Common-Mode Detector ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x0A, 0x07);
    // CMDET_EN: Enable CM detector on IN1, IN2, IN3 (RA, LA, LL)

    // --- Right Leg Drive (RLD) ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x0C, 0x04);
    // RLD_CN: Connect RLD output to IN4 (RL electrode) to reduce common-mode noise

    // --- Wilson Central Terminal (WCT) ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x0D, 0x01);
    // WILSON_EN1: Buffer 1 connected to IN1 (RA)

    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x0E, 0x02);
    // WILSON_EN2: Buffer 2 connected to IN2 (LA)

    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x0F, 0x03);
    // WILSON_EN3: Buffer 3 connected to IN3 (LL)
    // WCT = (RA + LA + LL) / 3 — used as reference for precordial leads on slaves

    // --- Clock ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x12, 0x05);
    // OSC_CN: Use external crystal + enable CLK output pin
    // Master outputs clock to slave modules via CLK pin

    // --- Channel Shutdown ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x14, 0x24);
    // AFE_SHDN_CN: Shut down CH3 INA and SDM (not used on master)

    // --- AFE Resolution ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x13, 0x00);
    // AFE_RES: All channels low-power mode, SDM clock 102.4 kHz

    // --- Digital Filter Decimation Rates ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x21, 0x04);
    // R2_RATE: R2 = 6 (sets PACE bandwidth)

    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x22, 0x04);
    // R3_RATE_CH1: R3 = 8 → ODR_ECG = 102400 / (4×6×8) = 533 Hz ≈ 522 Hz target

    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x23, 0x04);
    // R3_RATE_CH2: R3 = 8 (same as CH1)

    // --- DRDYB Source ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x27, 0x08);
    // DRDYB_SRC: DRDYB driven by CH1 ECG (fastest channel)
    // STM32 EXTI interrupt triggers on this pin

    // --- SYNCB (Master Output) ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x28, 0x08);
    // SYNCB_CN: SYNCB output driven by CH1 ECG
    // Synchronizes all 3 ADS1293 modules simultaneously

    // --- Loop Readback ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x2F, 0x30);
    // CH_CNFG: Enable CH1 ECG + CH2 ECG for streaming readback

    HAL_Delay(20);

    // --- Start Conversion ---
    writeRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x00, 0x01);
    // CONFIG: START_CON = 1 → begin ADC conversion
}
```

### Slave Modules (ADS1293 #2 & #3 — Precordial Leads V1–V6)

```c
void setup_ECG_Slave(GPIO_TypeDef* csPort, uint16_t csPin) {
    // --- Input Channel Routing ---
    writeRegister(csPort, csPin, 0x01, 0x0C);
    // FLEX_CH1_CN: CH1+ = IN1 (Vx), CH1- = IN4 (WCT from master)
    // Measures precordial lead: Vx - WCT

    writeRegister(csPort, csPin, 0x02, 0x14);
    // FLEX_CH2_CN: CH2+ = IN2 (Vx+1), CH2- = IN4 (WCT)

    writeRegister(csPort, csPin, 0x03, 0x1C);
    // FLEX_CH3_CN: CH3+ = IN3 (Vx+2), CH3- = IN4 (WCT)

    // --- Clock ---
    writeRegister(csPort, csPin, 0x12, 0x06);
    // OSC_CN: Use external clock from master CLK pin
    // Slave does not generate its own clock

    // --- AFE Resolution ---
    writeRegister(csPort, csPin, 0x13, 0x00);
    // AFE_RES: Low-power mode, SDM 102.4 kHz (same as master)

    // --- Digital Filter Decimation Rates ---
    writeRegister(csPort, csPin, 0x21, 0x04);
    // R2_RATE: R2 = 6

    writeRegister(csPort, csPin, 0x22, 0x04);
    // R3_RATE_CH1: R3 = 8 → same ODR as master (~522 Hz)

    writeRegister(csPort, csPin, 0x23, 0x04);
    // R3_RATE_CH2: R3 = 8

    writeRegister(csPort, csPin, 0x24, 0x04);
    // R3_RATE_CH3: R3 = 8

    // --- DRDYB ---
    writeRegister(csPort, csPin, 0x27, 0x00);
    // DRDYB_SRC: DRDYB not asserted by slave
    // Only master drives the DRDYB interrupt to STM32

    // --- SYNCB (Slave Input) ---
    writeRegister(csPort, csPin, 0x28, 0x40);
    // SYNCB_CN: DIS_SYNCBOUT = 1 → SYNCB pin configured as input
    // Slave receives synchronization signal from master

    // --- Loop Readback ---
    writeRegister(csPort, csPin, 0x2F, 0x70);
    // CH_CNFG: Enable CH1 + CH2 + CH3 ECG for streaming readback

    HAL_Delay(30);

    // --- Start Conversion ---
    writeRegister(csPort, csPin, 0x00, 0x01);
    // CONFIG: START_CON = 1
    HAL_Delay(10);
    writeRegister(csPort, csPin, 0x00, 0x01);
    // Second write ensures conversion starts cleanly after sync
}
```

### Sampling Frequency Calculation

The output data rate (ODR) for each ECG channel is determined by:

```
ODR_ECG = fS / (R1 × R2 × R3)

fS  = 102,400 Hz  (SDM clock, low-power mode)
R1  = 4           (standard PACE data rate, default)
R2  = 6           (register 0x21 = 0x04)
R3  = 8           (register 0x22–0x24 = 0x04)

ODR = 102400 / (4 × 6 × 8) = 533 Hz
```

---

## Integrating `ads1293.h` / `ads1293.c` into Your `main.c`

`main.c` itself is **not included as a file in this repo's instructions** — it's mostly auto-generated by STM32CubeIDE/CubeMX (clock tree, GPIO init, SPI init, MPU config), and that part is tied to your specific board/pin/clock setup, so it shouldn't be copy-pasted from someone else's project. Instead, generate your own `main.c` from CubeMX using the SPI / GPIO / Clock configuration described in the sections above, then add the following inside the existing `USER CODE` sections.

**1. Include the driver** — `/* USER CODE BEGIN Includes */`
```c
#include "ads1293.h"
```

**2. Add the data-ready flag** — `/* USER CODE BEGIN PV */`
```c
volatile uint8_t dataReady = 0;
```

**3. After peripheral init, inside `main()`** — `/* USER CODE BEGIN 2 */`
```c
// Verify all 3 ADS1293 modules respond
uint8_t id1 = readRegister(CS_ADS1_PORT, CS_ADS1_PIN, 0x40);
uint8_t id2 = readRegister(CS_ADS2_PORT, CS_ADS2_PIN, 0x40);
uint8_t id3 = readRegister(CS_ADS3_PORT, CS_ADS3_PIN, 0x40);
printf("ADS IDs: 1=0x%02X  2=0x%02X  3=0x%02X\r\n", id1, id2, id3);

HAL_Delay(100);
setup_ECG_Master();
HAL_Delay(20);
setup_ECG_Slave(CS_ADS2_PORT, CS_ADS2_PIN);
HAL_Delay(20);
setup_ECG_Slave(CS_ADS3_PORT, CS_ADS3_PIN);
HAL_Delay(20);
```
> Order matters here — the slaves rely on the clock and SYNCB signal already running from the master, so `setup_ECG_Master()` must always run first.

**4. Inside the main loop** — `/* USER CODE BEGIN WHILE */`
```c
if (dataReady) {
    dataReady = 0;
    readDataLoop();
}
```

**5. DRDYB interrupt callback** — anywhere in `/* USER CODE BEGIN 0 */ ... END 0 */`
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == DRDYB_Pin)
        dataReady = 1;
}
```
> Keep this callback minimal — it should only set the flag. The actual SPI read (`readDataLoop()`) runs in the main loop, not inside the ISR, so the interrupt handler returns immediately instead of blocking on SPI transfers.

> ⚠️ Everything else generated by CubeMX (`MPU_Config`, `SystemClock_Config`, `MX_SPI1_Init`, `MX_GPIO_Init`) should come from your own Pinout/Clock/Peripheral configuration in the STM32CubeIDE GUI, matching the tables earlier in this README — not copied verbatim from this repo.

---

## 📁 Repository Structure

```
ads1293-stm32-setup/
├── Core/
│   ├── Src/
│   │   ├── main.c
│   │   └── ads1293.c
│   └── Inc/
│       └── ads1293.h
├── docs/
│   ├── wiring.jpg
│   ├── spi_config.png
│   ├── gpio_config.png
│   ├── clock_config.png
│   └── pin_assignment.png
└── README.md
```

---

## 🚀 How to Run

1. Clone this repository:
```bash
git clone https://github.com/fairoozazzahra/ads1293-stm32-setup.git
```

2. Open the project in **STM32CubeIDE**

3. Configure STM32CubeIDE manually:
   - SPI1: Full-Duplex Master, Prescaler 128, CPOL Low, CPHA 1 Edge, 8-bit, NSS Disabled
   - GPIO PG0, PG1, PG3: Output Push Pull, default HIGH
   - GPIO PG2: EXTI Falling Edge, Pull-up (DRDYB interrupt)
   - Clock: HSE 8 MHz → PLL → SYSCLK 240 MHz → D1CPRE /2 → 120 MHz CPU/AHB/APB; SPI1,2,3 sourced from PLL1Q at 120 MHz

4. Build and flash to STM32 Nucleo-144 H743ZI2

5. Connect 3× ADS1293 modules according to the pin configuration above

6. Power on — master ADS1293 starts the clock and synchronizes all 3 modules via SYNCB. DRDYB interrupt fires when data is ready.

---

## 📚 References

- [ADS1293 Datasheet — Texas Instruments](https://www.ti.com/product/ADS1293)
---

## 👩‍💻 Author

**Fairooz Azzahra Darmawan**  
Electromedical Engineering, Poltekkes Kemenkes Surabaya  
fairoozazzahra1093@gmail.com
