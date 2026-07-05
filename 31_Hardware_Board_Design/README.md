# 31 — Hardware Board Design (SW Engineer's Guide)

> A software engineer who understands hardware is worth 3× a pure SW engineer. This section gives you the hardware literacy needed to collaborate effectively with HW teams and design simple boards yourself.

---

## Section Structure

```
31_Hardware_Board_Design/
├── 01_Schematic_Reading.md           ← How to read schematics as a SW engineer
├── 02_Power_Delivery.md              ← PMICs, LDOs, power sequencing, decoupling
├── 03_Clock_Distribution.md          ← Oscillators, PLLs, clock buffers
├── 04_High_Speed_Interfaces.md       ← PCIe, USB3, MIPI, signal integrity basics
├── 05_Memory_Interfaces.md           ← LPDDR4/5 layout, NAND/eMMC/UFS routing
├── 06_Debug_Connectors.md            ← JTAG, UART, USB test point design
├── 07_Simple_Board_Design.md         ← Design a minimal RK3588 carrier board
└── 08_HW_SW_Collaboration.md         ← How to work with hardware engineers effectively
```

---

## Schematic Reading for SW Engineers

### Key Symbols to Know

```
Component    Symbol     What it does
──────────── ────────── ──────────────────────────────────────
Resistor     ──/\/\/──  Current limiting, pull-up/pull-down
Capacitor    ──||──     Decoupling, bypass, filtering
Inductor     ──UUUU──   Power filtering, EMI suppression
Regulator    ──[REG]──  Convert voltage (3.3V from 5V)
Oscillator   ─[OSC]─    Generate clock signal
PMIC         ─[PMIC]─   Multiple regulators in one chip
```

### How to Read a Power Schematic

```
1. Find the main power input (VBAT, VIN, USB VBUS)
2. Trace through PMIC to individual rails (VCC3V3, VCC1V8, etc.)
3. Check power sequencing (some rails must come up before others)
4. Verify decoupling caps near SoC power pins (10µF + 100nF minimum)
5. Check enable signals (PMIC GPIO enables to SoC)
```

---

## SoC Power Rails (RK3588 Example)

```
VIN (5V USB-C / 12V DC)
│
├── PMIC (Rockchip RK806)
│   ├── VDD_CPU_BIG0 (0.75-1.0V) → Cortex-A76 big cores
│   ├── VDD_CPU_LIT (0.75-1.0V)  → Cortex-A55 LITTLE cores
│   ├── VDD_GPU (0.75-1.0V)      → Mali-G610 GPU
│   ├── VDD_NPU (0.75-1.0V)      → 6 TOPS NPU
│   ├── VCC_DDR (1.1V)           → LPDDR5 memory
│   ├── VCC3V3_SYS (3.3V)        → General 3.3V
│   └── VCC1V8_SYS (1.8V)        → General 1.8V
│
└── Discrete LDOs
    ├── VCC_3V3_SD (3.3V)         → SD card
    └── VCC_1V8_FLASH (1.8V)     → eMMC / QSPI flash
```

---

## Debug Connector Design

Always include these on custom boards:

```
UART Debug Header (4-pin, 2.54mm):
  Pin 1: GND
  Pin 2: 3.3V (optional, powers USB-UART adapter)
  Pin 3: UART TX (SoC transmit → host receive)
  Pin 4: UART RX (SoC receive ← host transmit)

JTAG/SWD Header (10-pin ARM standard):
  Pin 1: VREF (3.3V)
  Pin 2: SWDIO / TMS
  Pin 3: GND
  Pin 4: SWCLK / TCK
  Pin 5: GND
  Pin 6: SWO / TDO
  Pin 7: KEY (No Connect)
  Pin 8: TDI
  Pin 9: GND
  Pin 10: nRESET

Maskrom / Recovery Header:
  - Short TP_MASKROM to GND during power-on → enters BootROM download mode
  - Always expose this on prototype boards
```

---

## High-Speed Interface Signal Integrity

```
PCIe / USB3 / MIPI Design Rules:
1. Differential pairs must be length-matched (< 5 mil skew)
2. Maintain 100Ω differential impedance (trace width + gap)
3. No 90° corners — use 45° or curved bends
4. Keep reference ground plane continuous under traces
5. Minimize via stubs (backdrilling or HDI for Gen3+)
6. Place AC coupling caps within 200 mil of transmitter

For UFS 4.0 (your expertise):
- HS-Gear4 requires careful routing (2.9 Gbps per lane)
- Termination resistors must be accurate (±1%)
- EMI shield may be needed on long traces
```

---

## Interview Questions

| Level | Question |
|-------|----------|
| **Basic** | What is a pull-up resistor? When do you need one on an I2C bus? |
| **Intermediate** | How do you debug a power rail issue during board bring-up? |
| **Advanced** | What is signal integrity and why does it matter for PCIe/USB3? |
| **Expert** | Describe the power sequencing requirements for a Qualcomm SoC. What happens if the sequence is wrong? |
