# 6DOF Robotic Arm

A 6 degree-of-freedom robotic arm built from scratch. The project covers the full hardware and software stack: a custom STM32-based motor controller PCB, 3D-printed mechanical structure, and embedded firmware.

---

## Project Status

| Area | Status |
|---|---|
| Schematic | Complete |
| PCB Layout | In progress |
| STM32CubeMX pin assignment | Complete |
| Firmware | Not started |
| 3D Design | Not started |

---

## Repository Structure

```
.
├── LICENSE
├── PCB/
│   ├── Arm_Controller.kicad_sch       # Top-level schematic
│   ├── MCU.kicad_sch                  # STM32G431CBT6 schematic sheet
│   ├── Drivers.kicad_sch              # TMC2209 driver schematic sheet
│   ├── Power.kicad_sch                # Power supply schematic sheet
│   ├── Connectors.kicad_sch           # Connector schematic sheet
│   ├── Indicator.kicad_sch            # LED indicator schematic sheet
│   ├── Arm_Controller.kicad_pcb       # PCB layout
│   ├── Arm_Controller.kicad_pro       # KiCad project file
│   └── hardware/                      # Datasheets and reference material
└── STM32Cube/
    ├── Core/                          # Generated HAL application code
    ├── Drivers/                       # STM32 HAL and CMSIS drivers
    ├── EWARM/                         # IAR project files
    └── Robot Arm.ioc                  # STM32CubeMX configuration
```

---

## Hardware Overview

### Microcontroller — STM32G431CBT6

The board is centred around an STM32G431CBT6 in LQFP48 packaging, running at 170 MHz via PLL sourced from an 8 MHz external HSE crystal. USB uses the internal HSI48 oscillator via the CK48 mux to hit exactly 48 MHz without pulling the main PLL.

Flash latency is configured to `FLASH_LATENCY_4` as required at 170 MHz. The HAL timebase is moved from SysTick to TIM6 to avoid conflict with the USB stack.

SWD debug is available on PA13 (SWDIO) and PA14 (SWCLK).

### Stepper Motor Drivers — TMC2209

Six TMC2209 drivers handle the six stepper motor axes. Drivers are split into two banks of three, each sharing a UART line for configuration:

- **USART1 (PA9)** — Motors 1–3, half-duplex, 115200 baud
- **USART2 (PB3)** — Motors 4–6, half-duplex, 115200 baud

Up to three drivers share a single UART line using the TMC2209 address selection scheme via MS1/MS2 pins (addresses 0–3).

Each driver exposes a STEP input driven by hardware PWM (TIM2/TIM3 channels), a DIR GPIO, and a shared active-low enable line per bank (TMC_A_ENN, TMC_B_ENN). DIAG outputs from all six drivers are wired to EXTI lines 10–15 on PORTB for simultaneous independent fault interrupts.

Motor output pins (A1, A2, B1, B2 per driver) connect to 5 mm pitch screw terminals for direct stepper wiring.

Sense resistors are 0.11 Ω in 1206 package. Sustained motor current must not exceed 1.5 A per driver to stay within the 0.25 W resistor rating.

The TMC2209 accepts 4.75 V to 29 V on its motor supply. The 24 V rail is protected by a TVS diode clamping at ~29 V to keep transient spikes within this limit, particularly relevant when running from a 6S LiPo (max 25.2 V fully charged).

#### SPREAD Pin Configuration

Each driver has a dedicated 3-way solder jumper (`TMCX_SPREAD`) to select the chopper mode:

| SPREAD state | Mode |
|---|---|
| Pulled LOW (GND) | StealthChop2 — ultra-quiet operation |
| Pulled HIGH (3V3) | SpreadCycle — maximum torque |

Default: pulled HIGH (SpreadCycle). A 100 Ω series resistor is placed between the 3V3 rail and pad 1 of each jumper to limit current in the event of accidental full bridge.

> **Friendly Reminder:** Always have exactly ONE bridge between two pads! Keeping SPREAD floating will result in undefined behaviour and bridging all 3 pads will short GND and +3V3, resulting in a useless piece of green fiberglass. We do not take responsibility for this happening and recommend always testing for a short after resoldering these solder bridges.

### CAN Bus — FDCAN1 with ISO1042DWVR Isolator

CAN communication uses the STM32 FDCAN1 peripheral on PB8 (RX) and PB9 (TX), galvanically isolated via an ISO1042DWVR transceiver. The MCU side runs at 3.3 V; the bus side uses a 5 V isolated supply. Both supply pins carry decoupling capacitors close to the IC.

A 120 Ω termination resistor between CAN_H and CAN_L is present on the bus side, controlled by a solder bridge jumper (JP1). Close JP1 only if this board sits at a physical endpoint of the CAN bus. The default state is open.

> **Note:** JP1 default is OPEN. Exactly two 120 Ω terminations must be present across the entire CAN bus.

The ISO1042 is rated for CAN FD up to 5 Mbps. The board is currently configured for classic CAN. FDCAN timing parameters will need to be recalculated once SYSCLK is confirmed at 170 MHz.

> **TODO:** CAN bus connector footprint is `Connector_Molex:Molex_Pico-Lock_504050-0791_1x07-1MP_P1.50mm_Horizontal`. Exact pinout for the robot-side harness is not yet defined — document and lock this before PCB sign-off.

### Power System

Three 24 V power inputs feed the board, all ORed onto a shared 24 V bus:

- **Robot power distribution board** — dedicated 2-pin connector from the robot's main power distribution board. Protected by a Schottky diode so no current can ever flow back into the distribution board, regardless of what else is connected.
- **Battery** — XT60 female connector on-board, intended for 6S LiPo (22.2 V nominal, 25.2 V max fully charged). Protected by a Schottky diode in the same way.
- **J_PWR24V** — 2-pin 5 mm screw terminal. Accepts a standard 24 V DC supply (e.g. wall adapter with bare wires) or any external 24 V source. Also usable as a 24 V output tap (e.g. for a hotend heater). **No diode protection on this connector** — it connects directly to the 24 V bus. Polarity is marked clearly on silkscreen. Observe carefully: reverse polarity here will damage the board.

A fuse rated 15 A / 50 V is placed after the diodes, before the bus. Only one power source should be connected at a time.

#### 24 V Rail Protection

The 24 V rail carries bulk decoupling and transient protection sized for the worst-case load of 6 simultaneously switching stepper drivers:

- A large electrolytic bulk capacitor (1000 µF / 50 V) absorbs motor current transients
- A ceramic capacitor in parallel handles high-frequency switching noise
- A unidirectional TVS diode clamps transients to protect the TMC2209 drivers, which have a 29 V absolute maximum rating

#### 3.3 V Regulation

Two separate paths supply the 3.3 V rail depending on power source. Both feed through a Schottky diode OR arrangement so neither source can back-feed the other.

**Buck converter (24 V → 3.3 V), used during normal operation:**

The buck converter output is nominally ~3.65 V. With approximately 0.30 V drop across the downstream Schottky diode at typical load, the STM32 VDD rail sits at approximately 3.35 V — within the 2.0 V–3.6 V operating range.

**LDO (USB 5 V → 3.3 V), used during debug/programming only:**

The LDO output is 3.3 V. With the Schottky drop (~0.30 V at typical load), the STM32 VDD rail sits at approximately 3.0 V — still within the operating range, but marginal. The LDO is low-efficiency, which is acceptable for the limited debug use case. USB power does not supply the motor drivers.

> If exactly 3.3 V is required at the MCU, replace the Schottky diode with an ideal diode circuit, or remove the debug USB power path entirely.

#### 3.3 V Rail Filtering

Given the noisy 24 V environment (6 switching stepper drivers), the 3.3 V rail carries additional filtering beyond a basic dev board:

- A ferrite bead at the rail entry point attenuates noise before it reaches any VDD decoupling
- A bulk reservoir capacitor handles load transients and converter lag
- Three 100 nF ceramics provide per-pin HF decoupling on VDD
- A second, higher-impedance ferrite bead further isolates the VDDA analog supply from VDD
- VDDA carries both a 100 nF and a 1 µF capacitor per ST's reference design
- VREF+ carries a 100 nF decoupling capacitor; a 0 Ω jumper allows VREF+ to be separated from VDDA if needed

### USB

Full-speed USB device on PA11 (DM) and PA12 (DP), clocked from HSI48. Intended for firmware flashing and debug only — does not power the motor drivers.

---

## Pin Assignment

### STEP Outputs (Hardware PWM)

| Pin | Label | Timer Channel |
|---|---|---|
| PA0 | TMC1_STEP | TIM2_CH1 |
| PA1 | TMC2_STEP | TIM2_CH2 |
| PA6 | TMC3_STEP | TIM3_CH1 |
| PA7 | TMC4_STEP | TIM3_CH2 |
| PB0 | TMC5_STEP | TIM3_CH3 |
| PB1 | TMC6_STEP | TIM3_CH4 |

TIM2 and TIM3 run independently, allowing different step rates per motor bank.

### DIR Outputs (GPIO)

| Pin | Label |
|---|---|
| PA3 | M1_DIR |
| PA4 | M2_DIR |
| PA5 | M3_DIR |
| PB4 | M4_DIR |
| PB5 | M5_DIR |
| PB6 | M6_DIR |

Grouped by port to allow efficient single-register writes.

### Enable Outputs (Active LOW)

| Pin | Label | Controls |
|---|---|---|
| PA10 | TMC_A_ENN | Motors 1–3 |
| PA15 | TMC_B_ENN | Motors 4–6 |

Pulling both low implements a hardware e-stop across all six axes.

### DIAG Inputs (EXTI)

| Pin | Label | EXTI Line |
|---|---|---|
| PB10 | TMC1_DIAG | EXTI10 |
| PB11 | TMC2_DIAG | EXTI11 |
| PB12 | TMC3_DIAG | EXTI12 |
| PB13 | TMC4_DIAG | EXTI13 |
| PB14 | TMC5_DIAG | EXTI14 |
| PB15 | TMC6_DIAG | EXTI15 |

All DIAG lines sit on unique EXTI lines, so all six can fire simultaneously without arbitration. DIAG is driven HIGH by the TMC2209 on stall or fault detection.

### Indicators

| LED | Pin | Resistor | Color |
|---|---|---|---|
| STATUS_LED | PC13 | 1 kΩ | Red |
| POWER_LED | +3V3 (always-on) | 1.1 kΩ | Green |
| ACTIVITY_LED | PA8 | 390 Ω | Blue |

### Expansion Headers

| Pin | Use |
|---|---|
| PA2 | 3-pin 2.54 mm header (BTN/FAN/Thermistor) |
| PB2 | 3-pin 2.54 mm header (BTN/FAN) |
| PB7 | TIM4_CH2 — recommended for PWM fan control |

Headers are 3-pin 2.54 mm (GND / Signal / 3V3) with 10 kΩ pullup on the signal line for a defined state when unplugged.

PA2 is additionally suitable as a thermistor input (ADC) — the 10 kΩ pullup acts as the upper half of a voltage divider. Standard 10 kΩ NTC thermistors connect between the PA2 signal pad and GND.

---

## BOOT0 / PB8 — nBOOT_SEL Configuration

PB8 is shared between the FDCAN1_RX function and BOOT0. To free PB8 for CAN use permanently, the option byte **nBOOT_SEL = 1** must be programmed via STM32CubeProgrammer at first flash. This redirects boot mode selection away from the physical BOOT0 pin to the nBOOT0 option bit, which is set to 1 (normal boot from flash).

No physical BOOT or RESET button is fitted. NRST is accessible via pin 12 of the SWD header.

A test pad (TP_PB8) is placed on the PCB for two purposes:

1. Probing the CAN receive signal during bring-up.
2. Emergency bootloader recovery on boards where nBOOT_SEL was not programmed correctly — hold TP_PB8 HIGH at power-on to force the ROM bootloader via USB DFU.

---

## SWD Debug Header — J15

- Connector: 2×7 pin, 1.27 mm pitch, vertical through-hole
- KiCad footprint: `Connector_PinHeader_1.27mm:PinHeader_2x07_P1.27mm_Vertical`
- Pinout: STDC14 per ST UM2448 Table 6
- Preferred part: Samtec FTSH-107-01-L-DV-K-A (adds alignment shroud; any 2×7 1.27 mm header is electrically equivalent)
- Recommended debugger: **STLINK-V3MINIE** (ships with STDC14 ribbon cable)

| Pin | Signal |
|---|---|
| 1, 2 | Reserved — NC |
| 3 | T_VCC → +3V3 |
| 4 | SWDIO → PA13 |
| 5, 7 | GND |
| 6 | SWCLK → PA14 |
| 8 | SWO → NC (PB3 occupied by USART2) |
| 9 | NC |
| 10 | NC |
| 11 | GNDDetect → GND |
| 12 | NRST |
| 13, 14 | VCP RX/TX → NC |

---

## Test Points

All test points use 2.0 mm round pads for power rails and 1.0 mm round pads for signals, compatible with flying probe and pogo pin fixtures.

| Signal | Notes |
|---|---|
| GND | Power |
| +3V3 | Power |
| +24V | Power |
| VBUS | Power |
| CC1 | USB |
| CC2 | USB |
| TMC1–6 DIAG | One pad per driver |
| TMC1–6 INDEX | One pad per driver |
| TP_PB8 / BOOT0 | 1.5 mm round pad — CAN RX probe point and bootloader recovery |
| VDDA | Analog supply, after the VDDA ferrite bead |
| VREF+ | ADC reference rail |
| SWDIO | Debug |
| SWDCLK | Debug |
| NRST | Reset |
| CAN_H | Bus signal |
| CAN_L | Bus signal |
| CAN_GND | Bus ground reference |
| CAN_UART_TX | ISO1042 MCU-side TX |
| CAN_UART_RX | ISO1042 MCU-side RX |
| TMC_UART1 | USART1 half-duplex line (motors 1–3) |
| TMC_UART2 | USART2 half-duplex line (motors 4–6) |
| SW_NODE | Buck converter switching node — for scope noise analysis during bring-up |

---

## Known Issues and Open Items

- CAN connector pinout (Molex Pico-Lock 504050-0791) not yet defined for the robot-side harness.
- Connector for robot power distribution board not yet selected — footprint TBD.
- FDCAN timing parameters need recalculation after SYSCLK is confirmed at 170 MHz.
- PCB layout not started.
- Firmware not started.
- 3D mechanical design not started.

---

## Authorship

### 6DOF Robotic Arm — DBG-Robots

**Merlin Ortner**  
**Repository Owner & Principal Maintainer**  
Original author of this repository and responsible for long-term maintenance, development, and stewardship.  
Contact: ortnermerlin@gmail.com
