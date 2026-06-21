# 🌡️ STM32F407VGT6 TEMPERATURE-CONTROLLED FAN SPEED REGULATION

<div align="center">

![TU Chemnitz](https://img.shields.io/badge/TU%20Chemnitz-M.Sc.%20Embedded%20Systems-009640?style=for-the-badge)
![Embedded Systems](https://img.shields.io/badge/Project-Embedded%20Systems-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-In%20Progress-yellow?style=for-the-badge)
![License](https://img.shields.io/badge/License-Academic-orange?style=for-the-badge)

### 🛠️ Technical Skills

![Microcontroller](https://img.shields.io/badge/Microcontroller-STM32F407VGT6%20%7C%20Cortex--M4-black?style=for-the-badge)
![Embedded C](https://img.shields.io/badge/Embedded%20C-C%2FC%2B%2B-blue?style=for-the-badge)
![Sensor Integration](https://img.shields.io/badge/Sensor%20Integration-LM35%20Temperature%20Sensor-green?style=for-the-badge)
![HAL Driver Development](https://img.shields.io/badge/HAL%20Driver%20Development-STM32CubeIDE%20%7C%20CubeMX-orange?style=for-the-badge)
![Communication Protocols](https://img.shields.io/badge/Communication-UART%20%7C%20SPI%20%7C%20I2C%20%7C%20CAN-purple?style=for-the-badge)
![Control Systems](https://img.shields.io/badge/Control%20Systems-PID%20%7C%20PWM-red?style=for-the-badge)
![Debugging](https://img.shields.io/badge/Debugging-SWV%20ITM%20%7C%20Oscilloscope-yellow?style=for-the-badge)

> **Closed-loop thermal management using analog temperature sensing, PID control, and PWM-driven actuation — with breakpoint/register-level debugging and real-time SWV ITM telemetry.**

</div>

Closed-loop thermal management system built on the **STM32F407G-DISC1** Discovery Board (STM32F407VGT6, ARM Cortex-M4 @ 84 MHz), using an **LM35** analog temperature sensor, **PWM-driven MOSFET fan control**, **PID control logic**, and **GPIO LED thermal-zone indication**, with telemetry streamed over **SWV ITM / UART**.

> M.Sc. Embedded Systems, TU Chemnitz — May 2026
> Author: Rushikesh Lakadkar



---

## 📋 Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
- [Pin Assignment](#pin-assignment)
- [Complete System Schematic](#complete-system-schematic)
- [Wiring Diagram](#wiring-diagram)
- [Peripheral Configuration](#peripheral-configuration)
- [Control Logic](#control-logic)
- [PID Control Algorithm](#pid-control-algorithm)
- [Temperature Zones & LED Behavior](#temperature-zones--led-behavior)
- [Debugging Methodology & Verification](#debugging-methodology--verification)
- [Telemetry / UART Output](#telemetry--uart-output)
- [Demo Video](#demo-video)
- [Firmware](#firmware)
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
- Use STM32CubeIDE's SWV ITM Console for tracing, backed by breakpoint and register-level debugging

**Scope:** limited to ambient air temperature sensing and fan speed regulation on a breadboard prototype. Not designed for liquid cooling or high-power industrial loads, but the control architecture is directly transferable to such scenarios.

---

## Hardware

| Component | Spec |
|---|---|
| MCU | STM32F407VGT6 — ARM Cortex-M4, 168 MHz max (configured at 84 MHz), 1 MB Flash, 192 KB SRAM, LQFP-100 |
| Temperature Sensor | LM35DZ — 10 mV/°C, 0–100°C range, ±0.5°C accuracy, TO-92 |
| Fan | 5V DC brushless fan, MOSFET-switched |
| Fan Driver | N-channel MOSFET + 100 Ω gate resistor + flyback diode (1N4007 / SS14) |
| LEDs | 3× through-hole LEDs (Green/Yellow/Red) + 220–330 Ω resistors |
| IDE | STM32CubeIDE (HAL drivers, CubeMX peripheral config) |


> 📁 [`/STM32 Active board/`](./STM32_with_Active_Connections.jpeg/)

> 📁 [`/Complete Breadboard Prototype/`](./Breadboard_Prototype.jpeg/)




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

STM32CubeIDE's Pinout view confirms this exact assignment on the STM32F407VGTx (LQFP100) package — PD12–PD14 labelled `GPIO_Output`, PA0 as `ADC1_IN0`, PA2/PA3 as `USART2_TX`/`USART2_RX`, and PA5 as `TIM2_CH1`:

> 📁 [`/STM32CubeMX Pinout_/`](./STM32CubeIDE_Pinout_View.png/)

## Complete System Schematic

The diagram below consolidates every subsystem — the STM32F407G-DISC1 (U1), the LM35DZ sensor (U2), the MOSFET fan driver stage (U3), the LED indicator bank, the J1 Micro-USB connector, and the ST-LINK/UART breakout — into a single schematic.

> 📁 [`/Schematic_Wiring_/`](./Schematic_STM32_LM35_FAN_Project.png/)

**Power distribution:** the 3.3V rail from the Discovery board's on-board regulator supplies U1's logic and the LM35 (decoupled with a 10 µF bulk capacitor + 100 nF ceramic capacitor). The fan circuit (U3) runs from a separate +5V rail sourced from the USB bus via J1, keeping the fan's switching current off the 3.3V analog rail that feeds the LM35 — this is what keeps the ADC reading clean while the MOSFET switches at 1 kHz.

**Signal routing:** PA0 connects to the LM35's Vout with a 100 nF filter capacitor (C1) at the sensor node. PA5 (PWM) runs from TIM2_CH1 to the MOSFET gate (Q1) through the 100 Ω gate resistor (R1). PA2/PA3 (UART) branch to both the ST-LINK/UART breakout and the LED indicator bank's shared signal node.

**LED & fan stages:** PD12/13/14 each drive one LED through its own 330 Ω resistor (R2/R3/R4) to ground. The flyback diode D1 (SS14) sits across FAN1, clamping the inductive spike when Q1 (AO3400A N-channel MOSFET) switches off.

<p align="center">
  <img src="images/system-operation.jpg" width="320" alt="Assembled prototype during operation"/><br/>
  <em>Figure 5 — Assembled prototype during operation: breadboard wiring, LED indicators, and cooling fan</em>
</p>

## Wiring Diagram

![Wiring diagram](images/schematic.png)

*(see [Complete System Schematic](#complete-system-schematic) above for the full breakdown by subsystem)*

## Peripheral Configuration

| Peripheral | Setting |
|---|---|
| System Clock | HSI 16 MHz → PLL (M=16, N=336, P=DIV4) → 84 MHz SYSCLK, AHB /1, APB1 /2 (42 MHz, TIM2 clk = 84 MHz), APB2 /1 |
| ADC1 | 12-bit, single channel (PA0 / ADC_CHANNEL_0), software trigger, 3-cycle sampling, PCLK2/4 = 21 MHz ADC clock |
| TIM2 (PWM) | 1 kHz, 1000-step resolution (ARR = 999, Prescaler = 83), PWM Mode 1, Active High, Channel 1 (PA5) |
| USART2 | 115200 baud, 8N1, TX/RX, no flow control, 16x oversampling |
| GPIO (PD12–14) | Push-pull output, no pull-up/down, low speed |

STM32CubeIDE's Clock Configuration view confirms the full derivation path — HSI (16 MHz, blue) → Main PLL → 84 MHz SYSCLK → AHB Prescaler (/1) → 84 MHz HCLK, fanning out to APB1 (/2 → 42 MHz PCLK1, ×2 timer multiplier restores 84 MHz for TIM2) and APB2 (/1 → 84 MHz):

<p align="center">
  <img src="images/cubeide-clock-config.png" width="700" alt="STM32CubeIDE Clock Configuration view"/><br/>
  <em>Figure 6 — STM32CubeIDE Clock Configuration view: HSI → PLL → 84 MHz SYSCLK derivation and APB1/APB2 prescaler outputs</em>
</p>

## Control Logic

Main loop runs every **1 second**:

1. Trigger ADC conversion on PA0, poll for result
2. Convert ADC → voltage (`V = ADC × 3.3 / 4095`) → temperature (`T = V × 100`, since LM35 = 10 mV/°C)
3. Compute PID terms (P, I, D) against a 25.0°C setpoint
4. Clamp PID output to PWM range [0, 1000]
5. Override PID output with a discrete zone-based PWM value (see [Temperature Zones](#temperature-zones--led-behavior))
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

## PID Control Algorithm

A PID controller continuously calculates an error value as the difference between the desired setpoint and the measured temperature, then applies a correction based on three terms:

- **Proportional (P):** correction proportional to current error. Large Kp = fast response but possible overshoot.
- **Integral (I):** correction based on accumulated error over time. Eliminates steady-state offset but can cause windup if unmanaged.
- **Derivative (D):** correction based on rate of change of error. Damps overshoot, improves settling time.

```
output(n) = Kp × e(n) + Ki × SUM[e(n)] + Kd × [e(n) − e(n−1)]
```

| Parameter | Symbol | Value | Rationale |
|---|---|---|---|
| Setpoint | SP | 25.0 °C | Target ambient temperature for fan activation |
| Proportional Gain | Kp | 5.0 | Moderate gain; 1 °C error → 5 PWM units correction |
| Integral Gain | Ki | 0.1 | Slow integral wind-up, avoids oscillation |
| Derivative Gain | Kd | 1.0 | Moderate damping to reduce overshoot |
| Max PWM | pwm_max | 1000 | Maps to 100% duty cycle (TIM2 Period = 1000) |

> **PID vs. discrete zone override:** in the current prototype the PID output is computed and clamped every cycle, but fan speed is actually driven by the discrete zone override below (deterministic behavior for demo purposes). The PID's integral state still evolves throughout operation. Pure PID closed-loop control — using the PID output directly without the zone override — is planned as future work.

## Temperature Zones & LED Behavior

| Zone | Condition | PWM | Duty Cycle | LED |
|---|---|---|---|---|
| Low | T < 26°C | 300 | 30% | 🟢 Green |
| Medium | 27°C ≤ T ≤ 28°C | 600 | 60% | 🟡 Yellow |
| High | T > 28°C | 900 | 90% | 🔴 Red |

The active LED **toggles** (blinks at 0.5 Hz) rather than staying static, providing a visible "alive" indication. The other two LEDs are explicitly held low before the toggle.

```c
HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12|GPIO_PIN_13|GPIO_PIN_14, GPIO_PIN_RESET);
if (temperature < 26) {
  pwm_output = pwm_low; // 300
  HAL_GPIO_TogglePin(GPIOD, GPIO_PIN_12); // Green
} else if (temperature >= 27 && temperature <= 28) {
  pwm_output = pwm_mid; // 600
  HAL_GPIO_TogglePin(GPIOD, GPIO_PIN_13); // Yellow
} else {
  pwm_output = pwm_high; // 900
  HAL_GPIO_TogglePin(GPIOD, GPIO_PIN_14); // Red
}
```

> ⚠️ **Known issue:** an else-if boundary gap exists for 26–27°C (falls through to the High zone). Flagged for refactor with explicit hysteresis bands.

## Debugging Methodology & Verification

Beyond SWV ITM telemetry, development relied on STM32CubeIDE's integrated debugger (GDB over ST-LINK/V2 SWD) to verify peripheral behavior at the register level and catch logic errors before they surfaced as hardware faults.

**Breakpoint strategy** — four key points in `main.c`:
- After `HAL_ADC_GetValue()` — verify raw `adc_val`
- After the LM35 voltage→temperature conversion — confirm scaling
- Inside the PID block — inspect Kp/Ki/Kd contributions and clamped output
- Inside each zone branch — confirm correct branch and LED toggle

A conditional breakpoint (`temperature > 28.0f`) was used to catch the High-zone transition without manually heating the sensor on every run.

**Live variable inspection** (Variables view + Live Expressions):

| Variable | Typical Observed Value | What It Confirmed |
|---|---|---|
| `adc_val` | 2980 – 3010 | Stable 12-bit ADC code at room temperature |
| `voltage` | 2.40 – 2.42 | Voltage scaling formula matched expected LM35 output |
| `temperature` | 24.0 – 24.2 | LM35 ×100 scaling correct; agreed with reference thermometer |
| `error` / `integral` / `derivative` | varies | PID terms updating each iteration without runaway growth |
| `pwm_output` | 300 / 600 / 900 | Zone override correctly replacing the raw PID output |

**Peripheral register inspection** (SFR view, from the STM32F407VGT6 SVD file):

| Peripheral | Register | Field(s) Checked | Expected / Observed |
|---|---|---|---|
| ADC1 | SR | EOC | Set to 1 right after `HAL_ADC_PollForConversion()` returned |
| ADC1 | DR | DATA[11:0] | Matched `HAL_ADC_GetValue()` (e.g. 0xBB4 ≈ 2996) |
| ADC1 | CR2 | ADON, CONT | ADON = 1, CONT = 0 (single-conversion mode) |
| TIM2 | CCR1 | Capture/Compare 1 | Matched last value written via `__HAL_TIM_SET_COMPARE()` |
| TIM2 | ARR | Auto-reload value | 999, confirming the 1 kHz PWM period |
| TIM2 | CR1 | CEN | 1, confirming the timer was running |
| GPIOD | ODR | ODR12/13/14 | Only one bit high at a time, toggling at ~1 Hz |
| GPIOD | MODER | MODE12/13/14 | 01 (general-purpose output), confirming push-pull mode |

**SWV ITM console verification:** each printed telemetry line was cross-checked against the Variables view at the same breakpoint, and the 0.1 °C change-detection threshold was verified by watching the console during a stable-temperature period (no spurious lines).

**Oscilloscope cross-check:** TIM2_CH1 (PA5) was probed while halting on a breakpoint right after each zone's `__HAL_TIM_SET_COMPARE()` call:

| Zone | CCR1 (Register View) | Oscilloscope High Time | Period | Result |
|---|---|---|---|---|
| Low | 300 | ~300 µs | 1.00 ms (1 kHz) | ✅ Match |
| Medium | 600 | ~600 µs | 1.00 ms (1 kHz) | ✅ Match |
| High | 900 | ~900 µs | 1.00 ms (1 kHz) | ✅ Match |

This combination of breakpoints, register inspection, and ITM telemetry verification — cross-checked against the oscilloscope — directly caught three of the six issues in [Challenges & Solutions](#challenges--solutions) (missing `_write()` retarget, fast LED blinking, integral windup) before they were misattributed to a hardware fault.

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

📹 [System operation demo](videos/demo.mp4)

*(breadboard setup running live — fan spinning, LED zone indicators lit)*

## Firmware

📄 [`firmware/main.c`](firmware/main.c) — full application source: peripheral initialization (clock, ADC1, TIM2 PWM, USART2, GPIO), the 1-second control loop, PID computation, zone-override logic, and the SWV ITM `printf` retarget.

> Reconstructed from the peripheral configuration (Section 5) and code listings (Sections 6–8, 10) in the [full report](#full-report). The `MX_*_Init()` functions follow the documented CubeMX settings; re-export from the project's `.ioc` file in STM32CubeIDE for a byte-exact, build-verified copy.

```c
// Core control loop (runs every 1 second) — see firmware/main.c for the complete file
HAL_ADC_Start(&hadc1);
HAL_Delay(5);
HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
adc_val = HAL_ADC_GetValue(&hadc1);
voltage = (adc_val * 3.3f) / 4095.0f;
temperature = voltage * 100.0f;

float error = setpoint - temperature;
integral += error;
float derivative = error - prev_error;
pwm_output = Kp * error + Ki * integral + Kd * derivative;
if (pwm_output > pwm_max) pwm_output = pwm_max;
if (pwm_output < 0) pwm_output = 0;
prev_error = error;

__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, (uint32_t)pwm_output);
HAL_Delay(1000);
```

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
├── firmware/
│   └── main.c                      # Standalone copy of the application source (see Firmware section)
├── images/
│   ├── board.jpg                   # Discovery board photo
│   ├── breadboard.jpg              # Breadboard prototype photo
│   ├── system-operation.jpg        # Assembled prototype running (LEDs + fan)
│   ├── schematic.png               # Complete circuit schematic
│   ├── cubeide-clock-config.png    # STM32CubeIDE Clock Configuration screenshot
│   └── cubeide-pinout-view.png     # STM32CubeIDE Pinout view screenshot
├── videos/
│   └── demo.mp4                    # System operation demo
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
- **PWM verification:** oscilloscope confirmed 1 kHz signal, ~300 µs high period at 30% duty cycle, cross-checked against the TIM2_CCR1 register value (see [Debugging Methodology](#debugging-methodology--verification))

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

📄 [STM32_Temperature_Fan_Control_Report.pdf](docs/STM32_Temperature_Fan_Control_Report.pdf) — full write-up including system overview, PID derivation, peripheral configuration details, debugging methodology (breakpoints, register inspection, oscilloscope cross-checks), and complete test methodology.

---

*Built with STM32CubeIDE • HAL drivers • STM32F407G-DISC1*

<div align="center">

**Made with ⚙️ at Technische Universität Chemnitz**
*M.Sc. Embedded Systems*

</div>
