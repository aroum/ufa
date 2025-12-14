
# VXE R1 PRO Technical Specification

This document details the main active components, pinout, and circuit features of the VXE R1 PRO gaming mouse, intended for hardware modding and firmware development.

## 1. Key Integrated Circuits (ICs)

The table below lists the primary active components responsible for the mouse's core functionality (charging, power conversion, control, and sensor).

| **Component**             | **Model** | **Purpose**                               | **Notes**                                | **PCB Marking** | **Package** |
| ------------------------- | --------- | ----------------------------------------- | ---------------------------------------- | --------------- | ----------- |
| **Charger IC**            | TP4056    | Manages Li-Po battery charging process.   | Standard linear charge controller.       | U2              | SOP-8       |
| **DC-DC Converter**       | CXM2412   | Voltage step-down/conversion to 2.0 V.    | Supplies main voltage to MCU and sensor. | U5              | SOT-23-6    |
| **Microcontroller (MCU)** | nRF52840  | Main System Controller.                   | No external clock crystal observed.      | U?              | AQFN73      |
| **Optical Sensor**        | PWM3950   | High-performance optical tracking sensor. | Reset pin is not used                    | U7              | 16-PDIP     |

## 2. RGB LED Implementation

The mouse features two distinct RGB LED indicators: one for DPI status and one for charging status.

### 2.1. DPI LED (LED1) Components

This RGB LED is controlled via external transistor keys.

| **Sub-system**                 | **Value/Model** | **Package** | **PCB Marking** |
| ------------------------------ | --------------- | ----------- | --------------- |
| **DPI LED**                    | RGB             | SMD (0807)  | LED1            |
| **DPI LED LDO**                | CX33T (3.3V)    | SOT-23-5    | U6              |
| **DPI LED Resistor (Red)**     | 4.7 k           | SMD (0603)  | R30             |
| **DPI LED Resistor (Green)**   | 4.7 k           | SMD (0603)  | R31             |
| **DPI LED Resistor(Blue)**     | 4.7 k           | SMD (0603)  | R29             |
| **DPI LED Transistor (Red)**   | A1H             | SOT-23-3    | Q7              |
| **DPI LED Transistor (Green)** | A1H             | SOT-23-3    | Q8              |
| **DPI LED Transistor (Blue)**  | A1H             | SOT-23-3    | Q9              |

### 2.2. Charge LED (LED2) Components

| **Sub-system**                  | **Value/Model** | **Package** | **PCB Marking** |
| ------------------------------- | --------------- | ----------- | --------------- |
| **Charge LED**                  | RGB             | SMD (0807)  | LED2            |
| **Charge LED LDO**              | CX33T (3.3V)    | SOT-23-5    | U3              |
| **Charge LED Resistor (Red)**   | 240             | SMD (0603)  | R32             |
| **Charge LED Resistor (Green)** | 470             | SMD (0603)  | R34             |
| **Charge LED Resistor (Blue)**  | 470             | SMD (0603)  | R37             |

## 3. Controls and Input Elements

List of mechanical switches and the scroll wheel encoder used in the mouse.

| **Element**                | **Purpose**                           | **Package** | **PCB Marking** |
| -------------------------- | ------------------------------------- | ----------- | --------------- |
| **Power Switch**           | Mouse power ON/OFF toggle.            | MSK12C02    | K2              |
| **Encoder**                | Scroll Wheel mechanical encoder.      | 9mm height  | ENC             |
| **Left Button (Primary)**  | Main left click switch.               |             | LB              |
| **Right Button (Primary)** | Main right click switch.              |             | RB              |
| **Middle Button**          | Scroll wheel click switch.            |             | MB              |
| **DPI Button**             | Cycle through DPI sensitivity levels. |             | SW1             |
| **Side Button Forward**    | Auxiliary side button (Forward).      |             | FB              |
| **Side Button Back**       | Auxiliary side button (Back).         |             | BB              |

## 4. Power Management and Protection Components

This section lists components responsible for overvoltage protection and switching between USB and battery power sources.

| **Sub-system**          | **Component**  | **Marking** | **Package** | **PCB Marking** |
| ----------------------- | -------------- | ----------- | ----------- | --------------- |
| **VUSB Protection**     | Transistor     | HY3D        | SOT-23-3    | Q1              |
| **VUSB Protection**     | Transistor     | A1H         | SOT-23-3    | N1              |
| **Vbat Protection**     | Transistor     | HY4D        | SOT-23-3    | Q2              |
| **Vbat Protection**     | Transistor     | A1H         | SOT-23-3    | N3              |
| **Power Source Select** | Transistor     | A1H         | SOT-23-3    | N2              |
| **Power Source Select** | Schottky Diode | HSL         | SOD-123     | D1              |

## 5. Circuit Design Highlights

The circuit design of the VXE R1 Pro is electrically identical to the VGN F1 MOBA model.

| **Feature**                 | **Component**           | **Description**                                                                                                      |
| --------------------------- | ----------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Main Power Supply**       | DC/DC Converter #1      | Switching regulator with a fixed output voltage of **2.0 V** for the MCU and optical sensor.                         |
| **RGB LED Power**           | Linear Regulators (LDO) | Two dedicated LDOs (3.3 V output) are used to power the RGB LEDs.                                                    |
| **RGB LED Control**         | MCU Pins / Transistors  | The DPI LED (LED1) is controlled via external transistor keys (Q7-Q9). The Charge LED (LED2) uses direct/IC control. |
| **Encoder Debouncing**      | External Resistors      | External high-ohm pull-up/pull-down resistors are used on the encoder signal lines for stability.                    |
| **Charge Controller Logic** | TP4056                  | The charger's charge status (CHRG) signal is routed to a discrete input pin on the MCU for monitoring.               |
| **Battery Monitoring**      | Resistive Divider       | A voltage divider is connected to the MCU's ADC pin to monitor the current battery charge level.                     |
| **ESD Protection**          | Absent                  | There is no dedicated Electrostatic Discharge (ESD) protection circuitry observed on the board.                      |

## 6. Pinout Mapping (nRF52840 MCU)

The pinout is completely identical between the **VGN F1 MOBA** and **VXE R1 Pro** models.

| **Function**                        | **MCU Pin** |
| ----------------------------------- | ----------- |
| **Switches**                        |             |
| Left Button (LB)                    | P0.30       |
| Right Button (RB)                   | P0.29       |
| Middle Button (MB)                  | P0.28       |
| Forward Button (FB)                 | P0.03       |
| Back Button (BB)                    | P0.02       |
| DPI Button (SW1)                    | P1.15       |
| **Encoder**                         |             |
| Enc A                               | P0.00       |
| Enc B                               | P0.01       |
| **Sensor (SPI)**                    |             |
| Clock (SCK)                         | P0.06       |
| MOSI                                | P0.05       |
| MISO                                | P0.04       |
| Chip Select (CS)                    | P0.27       |
| Motion Interrupt Pin                | P0.07       |
| **Power and Charging**              |             |
| Battery Voltage Divider (ADC Input) | P0.31       |
| TP4056 CHRG Pin (Charge Status)     | P0.26       |
| **Charge LED Control**              |             |
| Charge LED LDO Enable               | P1.02       |
| Charger LED Red                     | P0.16       |
| Charger LED Green                   | P0.15       |
| Charger LED Blue                    | P0.14       |
| **DPI LED Control**                 |             |
| DPI LDO Enable                      | P1.01       |
| DPI LED Red (via Q7)                | P0.24       |
| DPI LED Green (via Q8)              | P0.22       |
| DPI LED Blue (via Q9)               | P0.21       |
