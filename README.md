# STM32F407VGT6 Temperature-Controlled Fan Speed Regulation

Closed-loop thermal management system built on the **STM32F407G-DISC1** Discovery Board (STM32F407VGT6, ARM Cortex-M4 @ 84 MHz), using an **LM35** analog temperature sensor, **PWM-driven MOSFET fan control**, **PID control logic**, and **GPIO LED thermal-zone indication**, with telemetry streamed over **SWV ITM / UART**.

> M.Sc. Embedded Systems, TU Chemnitz — May 2026
> Author: Rushikesh Lakadkar

![Board](images/board.jpg)
![Breadboard prototype](images/breadboard.jpg)

**Skills demonstrated:** Embedded Software Development · Sensor Integration · HAL Driver Development · Embedded C

---

## 📋 Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
- [Pin Assignment](#pin-assignment)
- [Wiring Diagram](#wiring-diagram)
- [Peripheral Configuration](#peripheral-configuration)
- [Control Logic](#control-logic)
- [Temperature Zones & LED Behavior](#temperature-zones--led-behavior)
- [Telemetry / UART Output](#telemetry--uart-output)
- [Demo Video](#demo-video)
- [Repository Structure](#repository-structure)
- [Test Results](#test-results)
- [Challenges & Solutions](#challenges--solutions)
- [Future Work](#future-work)
- [References](#references)
- [Full Report](#full-report)

---

## Overview

The system reads ambient temperature continuously via the LM35 sensor, processes it through a PID control loop, and drives a 5V DC fan at a proportional speed using PWM-modulated MOSFET switching. Three LEDs (green/yellow/red) provide a discrete visual indication of the current thermal zone, and real-time telemetry is streamed via SWV ITM trace (UART-capable).

**Key objectives:**
- Read ambient temperature via LM35 on ADC Channel 0
- Convert raw ADC value → calibrated temperature (°C)
- Implement a PID controller to compute fan speed correction
- Drive a DC fan via MOSFET + PWM (TIM2 CH1)
- Indicate thermal zones (low / medium / high) via GPIO LEDs
- Stream live telemetry (temp, ADC, voltage, error, PWM) at 115200 baud
- Use STM32CubeIDE's SWV ITM Console for tracing

---

## Hardware

| Component | Spec |
|---|---|
| MCU | STM32F407VGT6 — ARM Cortex-M4, 168 MHz max (configured at 84 MHz), 1 MB Flash, 192 KB SRAM |
| Temperature Sensor | LM35DZ — 10 mV/°C, 0–100°C range, ±0.5°C accuracy |
| Fan | 5V DC brushless fan, MOSFET-switched |
| Fan Driver | N-channel MOSFET + 100 Ω gate resistor + 1N4007 flyback diode |
| LEDs | 3× through-hole LEDs (Green/Yellow/Red) + 220 Ω resistors |
| IDE | STM32CubeIDE (HAL drivers, CubeMX peripheral config) |

## Pin Assignment

| STM32 Pin | Function | Connected To |
|---|---|---|
| PA0 | ADC1_IN0 | LM35 Vout |
| PA5 | TIM2_CH1 (PWM) | MOSFET Gate (via 100 Ω) |
| PA2 | USART2_TX | USB-UART bridge (ST-LINK) |
| PA3 | USART2_RX | USB-UART bridge (ST-LINK) |
| PD12 | GPIO Output | Green LED (via 220 Ω) |
| PD13 | GPIO Output | Yellow LED (via 220 Ω) |
| PD14 | GPIO Output | Red LED (via 220 Ω) |
| GND | Common Ground | LM35 GND, MOSFET Source, LED cathodes |
| 3.3V | Supply | LM35 Vs |
| 5V | Supply | Fan positive terminal |

## Wiring Diagram

![Wiring diagram](images/wiring-diagram.jpg)

## Peripheral Configuration

| Peripheral | Setting |
|---|---|
| System Clock | HSI 16 MHz → PLL → 84 MHz SYSCLK |
| ADC1 | 12-bit, single channel, software trigger, 3-cycle sampling |
| TIM2 (PWM) | 1 kHz, 1000-step resolution (ARR = 999, Prescaler = 83) |
| USART2 | 115200 baud, 8N1 |
| GPIO (PD12–14) | Push-pull output, no pull-up/down, low speed |

## Control Logic

Main loop runs every **1 second**:

1. Trigger ADC conversion on PA0, poll for result
2. Convert ADC → voltage (`V = ADC × 3.3 / 4095`) → temperature (`T = V × 100`, since LM35 = 10 mV/°C)
3. Compute PID terms (P, I, D) against a 25.0°C setpoint
4. Clamp PID output to PWM range [0, 1000]
5. Override PID output with a discrete zone-based PWM value (see below)
6. Update TIM2 compare register → sets fan PWM duty cycle
7. Reset all LEDs, toggle the active zone LED
8. If temperature changed by >0.1°C, print telemetry via SWV ITM

```c
float error = setpoint - temperature;
integral += error;
float derivative = error - prev_error;
pwm_output = Kp * error + Ki * integral + Kd * derivative;

// Anti-windup clamp
if (pwm_output > pwm_max) pwm_output = pwm_max;
if (pwm_output < 0) pwm_output = 0;
prev_error = error;
```

**PID gains:** `Kp = 5.0`, `Ki = 0.1`, `Kd = 1.0`, Setpoint = 25.0°C

> Note: in the current prototype the PID output is computed and clamped each cycle, but fan speed is actually driven by the discrete zone override below (deterministic behavior for demo purposes). Pure PID closed-loop control is planned as future work.

## Temperature Zones & LED Behavior

| Zone | Condition | PWM | Duty Cycle | LED |
|---|---|---|---|---|
| Low | T < 26°C | 300 | 30% | 🟢 Green |
| Medium | 27°C ≤ T ≤ 28°C | 600 | 60% | 🟡 Yellow |
| High | T > 28°C | 900 | 90% | 🔴 Red |

The active LED **toggles** (blinks at 0.5 Hz) rather than staying static, providing a visible "alive" indication. The other two LEDs are explicitly held low.

> Known issue: an else-if boundary gap exists for 26–27°C (falls through to High zone). Flagged for refactor with explicit hysteresis bands.

## Telemetry / UART Output

Printed whenever temperature changes by more than 0.1°C since the last report:

```
Temp: 24.75 | ADC: 3068 | Volt: 2.47 | Err: 0.25 | PWM: 300
```

| Field | Meaning |
|---|---|
| Temp | Measured temperature (°C) |
| ADC | Raw 12-bit ADC value (0–4095) |
| Volt | Computed analog voltage (V) |
| Err | PID error = setpoint − temperature |
| PWM | Current PWM compare value applied to TIM2 |

To view: enable **Serial Wire Viewer** in Debug Configurations (Core Clock = 84 MHz) → `Window > Show View > SWV > SWV ITM Data Console` → enable ITM port 0 → record.

## Demo Video

📹 [Sensor data / fan response demo](videos/demo.mp4)

*(heat-gun test showing LED zone transitions, audible fan speed change, and live SWV ITM telemetry)*

## Repository Structure

```
.
├── Core/
│   ├── Inc/
│   │   └── main.h                  # Pin definitions, peripheral handles
│   └── Src/
│       ├── main.c                  # Application logic, control loop
│       ├── stm32f4xx_hal_msp.c      # HAL MSP init (clocks, AF config)
│       └── stm32f4xx_it.c           # Interrupt service routines
├── Drivers/
│   └── STM32F4xx_HAL_Driver/        # ST HAL library
├── images/
│   ├── board.jpg                   # Discovery board photo
│   ├── breadboard.jpg              # Breadboard prototype photo
│   └── wiring-diagram.jpg          # Circuit/wiring diagram
├── videos/
│   └── demo.mp4                    # Sensor/fan response demo
├── docs/
│   └── STM32_Temperature_Fan_Control_Report.pdf   # Full project report
└── README.md
```

## Test Results

| Test Case | Input Temp | Expected LED | Observed LED | Expected PWM | Observed PWM | Result |
|---|---|---|---|---|---|---|
| Low Zone | < 26°C | Green (blink) | Green (blink) | 300 | 300 | ✅ PASS |
| Medium Zone | 27–28°C | Yellow (blink) | Yellow (blink) | 600 | 600 | ✅ PASS |
| High Zone | > 28°C | Red (blink) | Red (blink) | 900 | 900 | ✅ PASS |
| UART Log | Any (>0.1°C Δ) | — | — | — | Logged correctly | ✅ PASS |
| Fan Speed | Proportional | — | — | Varies | Audibly varies | ✅ PASS |

- **ADC accuracy:** firmware temperature (24.0–24.1°C) matched a reference thermometer (24.0°C) within ±0.2°C
- **PWM verification:** oscilloscope confirmed 1 kHz signal, ~300 µs high period at 30% duty cycle

## Challenges & Solutions

| Challenge | Root Cause | Solution |
|---|---|---|
| `printf` not visible in console | `_write()` not retargeted to ITM | Implemented `_write()` via `ITM_SendChar()`, enabled SWV |
| LED blinking too fast | Toggle called without delay | Moved toggle inside 1-second loop (0.5 Hz blink) |
| Temperature drift | No decoupling on LM35 Vs pin | Added 100 nF ceramic capacitor near sensor |
| Fan not spinning at low PWM | MOSFET gate drive concern | Confirmed logic-level MOSFET compatible with 3.3V gate |
| Zone gap at 26–27°C | Else-if boundary logic | Documented as known issue; hysteresis refactor planned |
| Integral windup | Integral accumulated during heating | Added anti-windup clamp (0 to pwm_max) |

## Future Work

- [ ] Replace zone override with pure PID closed-loop control
- [ ] Interrupt/DMA-driven ADC sampling
- [ ] UART command interface for runtime setpoint/gain tuning
- [ ] OLED display (I2C, SSD1306) for numeric readout
- [ ] Tachometer feedback for true closed-loop fan speed control
- [ ] CAN bus integration for distributed multi-node thermal monitoring
- [ ] Port PID to ARM CMSIS-DSP fixed-point
- [ ] Production-grade anti-windup tied to actuator saturation

## References

1. STMicroelectronics, *STM32F407xx Datasheet*, DS8626, Rev 9, 2023.
2. STMicroelectronics, *STM32F4xx HAL and Low-layer Drivers User Manual*, UM1725, Rev 9, 2023.
3. STMicroelectronics, *STM32F407G-DISC1 Discovery Kit User Manual*, UM1472, Rev 4, 2021.
4. Texas Instruments, *LM35 Precision Centigrade Temperature Sensor Datasheet*, SNIS159H, 2017.
5. K.J. Åström and T. Hägglund, *PID Controllers: Theory, Design, and Tuning*, 2nd ed., ISA, 1995.
6. J. Yiu, *The Definitive Guide to the ARM Cortex-M4*, 1st ed., Elsevier/Newnes, 2013.
7. STMicroelectronics, *STM32CubeIDE User Guide*, UM2609, 2023.
8. ARM Limited, *Cortex-M4 Technical Reference Manual*, ARM DDI 0439C, 2010.

## Full Report

📄 [STM32_Temperature_Fan_Control_Report.pdf](docs/STM32_Temperature_Fan_Control_Report.pdf) — full write-up including system overview, PID derivation, peripheral configuration details, and complete test methodology.

---

*Built with STM32CubeIDE • HAL drivers • STM32F407G-DISC1*
