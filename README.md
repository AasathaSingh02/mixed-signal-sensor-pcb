# Mixed-Signal Sensor Data Acquisition Board

<p align="center">
  <img src="https://img.shields.io/badge/Platform-STM32F103-blue?style=flat-square&logo=stmicroelectronics" />
  <img src="https://img.shields.io/badge/EDA-KiCad%207-blue?style=flat-square&logo=kicad" />
  <img src="https://img.shields.io/badge/Simulation-LTSpice-orange?style=flat-square" />
  <img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" />
</p>

A **4-layer mixed-signal PCB** designed for industrial condition monitoring, integrating an STM32F103C8T6 microcontroller with PT100 RTD temperature sensing and ACS712 hall-effect current measurement. The analog front-end achieves **72 dB SNR** through careful grounding, filtering, and signal conditioning — all validated in LTSpice before layout.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ANALOG DOMAIN                                     │
│                                                                          │
│  PT100 RTD ──► Wheatstone ──► INA128 ──► Butterworth ──► ADS1115 ──┐    │
│               Bridge        (G=100)    LPF (fc=500Hz)   16-bit ADC  │    │
│                                                                      │    │
│  ACS712 ────────────────────► Butterworth ──────────────► ADS1115 ──┤    │
│  (±5A)                       LPF (fc=500Hz)              Ch.2       │    │
│                                                                      │    │
│ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ Star Ground ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤    │
│                                                                      │    │
│                        DIGITAL DOMAIN                                │    │
│                                                                      │    │
│  STM32F103 ◄════ SPI (1 MHz) ════════════════════════════════════════┘    │
│      │                                                                    │
│      ├──► UART (115200 baud) ──► USB-TTL ──► PC Terminal                  │
│      ├──► Status LEDs (Sampling, Error, Power)                            │
│      └──► GPIO: Calibration trigger, mode select                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Key Specifications

| Parameter | Value |
|-----------|-------|
| MCU | STM32F103C8T6 (ARM Cortex-M3, 72 MHz) |
| ADC | ADS1115 (16-bit, 860 SPS, I²C/SPI) |
| Temperature Sensor | PT100 RTD (Class A, ±0.15°C @ 0°C) |
| Current Sensor | ACS712-05B (±5A, 185 mV/A) |
| Signal Conditioning | INA128 instrumentation amp (CMRR: 120 dB) |
| Anti-Aliasing Filter | 2nd-order Sallen-Key Butterworth, fc = 500 Hz |
| SNR | 72 dB (measured at ADC input) |
| PCB Layers | 4 (Signal-GND-Power-Signal) |
| Communication | SPI (ADC), UART (debug @ 115200) |
| Power Supply | 3.3V (digital) / 5V (analog), LDO regulated |

## Directory Structure

```
mixed-signal-sensor-pcb/
├── firmware/
│   ├── inc/
│   │   ├── main.h
│   │   ├── ads1115.h          # ADS1115 ADC driver (SPI)
│   │   ├── pt100.h            # PT100 RTD temperature conversion
│   │   ├── acs712.h           # ACS712 current sensor processing
│   │   ├── filter.h           # Digital filtering (moving average, IIR)
│   │   └── uart_debug.h       # UART debug output
│   ├── src/
│   │   ├── main.c             # Main application loop
│   │   ├── ads1115.c          # ADS1115 SPI driver implementation
│   │   ├── pt100.c            # RTD linearization & calibration
│   │   ├── acs712.c           # Current measurement & offset calibration
│   │   ├── filter.c           # Digital filter implementations
│   │   └── uart_debug.c       # Formatted serial output
│   └── Makefile
├── simulation/
│   ├── analog_frontend.asc    # LTSpice: Complete analog front-end
│   ├── butterworth_lpf.asc    # LTSpice: Anti-aliasing filter
│   ├── ina128_gain_stage.asc  # LTSpice: Instrumentation amp stage
│   └── results/
│       └── simulation_report.md
├── hardware/
│   ├── bom.csv                # Bill of Materials
│   ├── design_rules.md        # PCB design rules & constraints
│   └── stackup.md             # 4-layer stackup specification
├── docs/
│   ├── design_notes.md        # Detailed design decisions
│   └── test_procedure.md      # Testing & validation procedure
├── test/
│   ├── snr_measurement.py     # SNR calculation from sampled data
│   └── calibration.py         # Sensor calibration scripts
├── LICENSE
└── README.md
```

## Analog Front-End Design

### PT100 Signal Chain
The PT100 RTD is placed in a **Wheatstone bridge** (R1=R2=R3=1kΩ, R_PT100 ≈ 100Ω at 0°C) excited by a precision 2.5V reference. The differential bridge output (~0.5 mV/°C) feeds an **INA128 instrumentation amplifier** set to G=100 (R_G = 505Ω), producing ~50 mV/°C.

The amplified signal passes through a **2nd-order Sallen-Key Butterworth low-pass filter** (fc = 500 Hz) before reaching the ADS1115. The Butterworth topology was chosen for its maximally flat passband — critical for accurate DC measurement.

### ACS712 Signal Chain
The ACS712 outputs a ratiometric voltage centered at VCC/2 (2.5V) with 185 mV/A sensitivity. A **DC-blocking capacitor + bias restoration** circuit centers the signal for the ADC's input range, followed by an identical Butterworth filter stage.

### Grounding Strategy
```
           ┌─────────────┐
           │  Star Ground │ (Single connection point near ADC)
           │    Point     │
           └──────┬───────┘
                  │
        ┌─────────┴─────────┐
        │                   │
   ┌────┴────┐         ┌───┴────┐
   │ Analog  │         │Digital │
   │ Ground  │         │Ground  │
   │ Plane   │         │Plane   │
   │(Layer 2)│         │(Layer 2)│
   └─────────┘         └────────┘
```

## Simulation Results

### Butterworth Filter Response
- **Passband ripple**: < 0.1 dB (0–400 Hz)
- **-3 dB frequency**: 498 Hz (design target: 500 Hz)
- **Stopband attenuation**: -40 dB/decade (2nd order)
- **Phase at fc**: -90° (as expected for Butterworth)

### SNR Analysis
- **Signal power** (1 Hz temperature variation): -18 dBV
- **Noise floor** (integrated 10 Hz – 10 kHz): -90 dBV
- **Measured SNR**: 72 dB

## Building the Firmware

### Prerequisites
- ARM GCC Toolchain (`arm-none-eabi-gcc`)
- STM32CubeMX (for HAL configuration)
- ST-Link programmer

### Compile & Flash
```bash
cd firmware/
make clean && make
make flash    # Requires ST-Link connected
```

### Serial Monitor
```bash
# Linux/Mac
screen /dev/ttyUSB0 115200

# Windows
# Use PuTTY or Tera Term on the appropriate COM port
```

## References
- [INA128 Datasheet](https://www.ti.com/lit/ds/symlink/ina128.pdf) — Texas Instruments
- [ADS1115 Datasheet](https://www.ti.com/lit/ds/symlink/ads1115.pdf) — Texas Instruments
- [ACS712 Datasheet](https://www.allegromicro.com/en/products/sense/current-sensor-ics/zero-to-fifty-amp-integrated-conductor-sensor-ics/acs712) — Allegro MicroSystems
- IPC-2221B — Generic Standard on Printed Board Design
- [KiCad 7 Documentation](https://docs.kicad.org/)

## License
MIT License — See [LICENSE](LICENSE) for details.
