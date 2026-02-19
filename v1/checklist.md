# Smartwatch Hardware Final Engineering Checklist

## 1. Power Regulation (TPS63000 Buck-Boost)
- [ ] **PS/SYNC Control:** Pin 7 (PS/SYNC) must be connected to an **RTC-capable GPIO** (e.g., GPIO 32 or 33) to maintain state during Deep Sleep.
- [ ] **Default Efficiency:** Ensure **R21 (100kΩ pull-down)** is populated on the PS/SYNC line so the regulator defaults to Power Save Mode (PFM) if the MCU pin floats.
- [ ] **Inductor Spec:** L1 must be **Sunlord SWPA4030S2R2MT** (2.2µH, ~3.35A Saturation) to handle high inrush currents without core saturation.
- [ ] **Input Filtering:** C8 (10uF) and C5 (0.1uF) must be placed immediately adjacent to the VIN and VINA pins.
- [ ] **Feedback Loop:** Keep the trace from R9/R10 to FB (Pin 10) as short as possible and physically isolated from the switching nodes (L1/L2).

## 2. Decoupling & Brownout Prevention
- [ ] **Regulator Local Caps:** C6/C7 (10uF each) placed at the TPS63000 VOUT.
- [ ] **ESP32 Bulk Reservoir:** Add a **47µF (10V or 16V X7R)** capacitor as close as possible to the ESP32 VDD pins to buffer 1s BLE wakeup spikes.
- [ ] **High-Frequency Bypass:** Ensure a **0.1µF** ceramic cap is paired with the 47µF cap at the ESP32 power entry point.
- [ ] **Trace Width:** Set a Net Class in KiCad for `3.3V_PWR` and `VBAT` with a minimum width of **0.5mm (20 mils)** to minimize impedance.

## 3. Vampire Drain Isolation (Leakage Control)
- [ ] **CH340N Isolation:** Verify the dual-channel MOSFET circuit is implemented on the TX/RX lines to prevent the ESP32 from "back-powering" the CH340N when USB is disconnected.
- [ ] **USB VBUS Sensing:** Ensure the MOSFET gate for CH340N isolation is triggered by the presence of **AUTO_SWITCH_5V**.
- [ ] **Battery Measurement:** The voltage divider for battery sensing must be gated by the **SI1016X MOSFET**.
- [ ] **Divider Control:** Ensure the GPIO controlling the SI1016X defaults to **LOW** (Off) to prevent constant ~200µA drainage.

## 4. Peripheral Integration & Logic
- [ ] **BMI270 Interrupts:** Route INT1/INT2 to **RTC GPIOs** (GPIO 32-39). This is mandatory for "Wake-on-Motion" from Deep Sleep.
- [ ] **GT911 Touch Reset:** Connect the GT911 Reset pin to a dedicated GPIO. Pulling this LOW is the "hard" backup for the I2C Sleep (0x05) command.
- [ ] **Strapping Pin Audit:** Double-check that none of the following are used for critical pull-ups/peripherals that could interfere with boot:
    - GPIO 0 (Boot mode)
    - GPIO 2 (Must be low for flashing)
    - GPIO 5 (Timing/Boot)
    - GPIO 12 (Flash Voltage - **CRITICAL**)
    - GPIO 15 (Silent logging)

## 5. Thermal & Grounding
- [ ] **Thermal Vias:** Place at least 4-6 vias in the Thermal Pad (GND) of the TPS63000 and the ESP32 to sink heat into the internal ground plane.
- [ ] **Ground Plane:** Maintain a solid, unbroken Ground Plane under the Power Stage and the ESP32 antenna area (ensure the antenna itself has the required keep-out clearance).

## 6. Firmware Duty-Cycle Targets (README)
- [ ] **WiFi:** Permanently disabled in Zephyr `prj.conf` (0mA).
- [ ] **BLE Background:** Set `bt_conn_le_param_update` to **1000ms interval**.
- [ ] **LVGL Task:** Must be suspended/aborted on Core 1 when screen times out to enable SoC Light Sleep.
- [ ] **Target Life:** 80 Hours (3.35 Days) based on 680mAh usable capacity.

***********************************************************************************************


# Hardware Design Specification: ESP32-WROOM-32E System

## 1. System Overview
*   **Module:** ESP32-WROOM-32E (ECO V3).
*   **Configuration:** Left-side clustering for SPI Display and RTC Crystal; Right-side for I2C and ADC.
*   **Operating Mode:** Wi-Fi permanently **Disabled** to allow ADC2 operation on Pin 26.

## 2. Master Pin Mapping Table


| Physical Pin | Logic Label | Primary Function | Hardware Type | Critical Notes |
| :--- | :--- | :--- | :--- | :--- |
| **6** | IO34 | **RTC Wakeup 1** | Input Only | **No Internal Pull-up.** External 10kΩ resistor required. |
| **7** | IO35 | **RTC Wakeup 2** | Input Only | **No Internal Pull-up.** External 10kΩ resistor required. |
| **8** | IO32 | **RTC Crystal P** | XTAL_32K | Connect to External 32.768kHz Crystal. |
| **9** | IO33 | **RTC Crystal N** | XTAL_32K | Connect to External 32.768kHz Crystal. |
| **10** | IO25 | **Free Digital Out**| Digital I/O | Available for Backlight, LED, or Buzzer. |
| **11** | IO26 | **Display RESET** | Digital Out | Hardware reset line for the SPI Display. |
| **12** | IO27 | **Display DC** | Digital Out | Data/Command Control for SPI Display. |
| **13** | IO13 | **SPI MOSI** | HSPI Bus | Connect to Display SDI/MOSI. |
| **14** | IO14 | **SPI CLK** | HSPI Bus | Connect to Display SCK/CLK. |
| **15** | IO15 | **SPI CS** | HSPI Bus | **Strapping Pin.** (MTDO) Must be LOW at boot. |
| **16** | IO12 | **SPI MISO** | HSPI Bus | **Strapping Pin.** (MTDI) Keep LOW/Float for 3.3V Flash. |
| **25** | IO0 | **Boot Button** | Strapping Pin | Pull-up to 3.3V; Momentary Button to GND for Flash. |
| **26** | IO4 | **ADC Sensor** | **ADC2_CH0** | **Strapping Pin.** Requires Wi-Fi OFF for operation. |
| **33** | IO21 | **I2C SDA** | Digital I/O | Data line for BMI270 and other I2C slaves. |
| **36** | IO22 | **I2C SCL** | Digital I/O | Clock line for BMI270 and other I2C slaves. |
| **34** | RXD0 | **UART RX** | Serial | Programming and Debugging interface. |
| **35** | TXD0 | **UART TX** | Serial | Programming and Debugging interface. |

## 3. Critical Hardware Checkpoints

### ⚠️ Boot Strapping & Flash Voltage
*   **Pin 16 (GPIO 12):** Sets internal flash voltage. Ensure the SPI Display does **not** drive MISO high during power-up. If high at boot, the ESP32 sets flash to 1.8V, causing a boot loop.
*   **Pin 26 (GPIO 4):** Ensure your analog sensor output does not hold a logic level that disrupts the boot sequence at power-on.

### ⚠️ External Pull-up Requirements
*   **Pins 6 & 7 (GPIO 34/35):** These are **Input Only** and lack internal pull-up/down resistors. You **must** add external 10kΩ pull-up resistors to 3.3V for reliable wakeup triggering.
*   **I2C (Pins 33/36):** While the ESP32 has internal pull-ups, external 4.7kΩ resistors are highly recommended when using multiple I2C slaves.

### ⚠️ ADC2 & Wi-Fi Conflict
*   **Pin 26 (ADC2_CH0):** Hardware-locked while Wi-Fi is active. Wi-Fi must be explicitly disabled in firmware to read the analog sensor.

### ⚠️ Crystal Layout
*   **Pins 8 & 9 (GPIO 32/33):** Route crystal traces as short as possible. Use a ground guard ring to isolate them from high-speed SPI signals (Pin 14) to prevent clock jitter.

**************************************************************************************************

Need to install Espressif library from plugin manager usin the zip in ext_kicad_models
