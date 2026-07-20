# ⚡ UPDI HV Programmer & Power Profiler Companion

Modern ATtiny (and newer AVR) microcontrollers allow the dedicated UPDI programming pin to be repurposed as a standard GPIO or UART serial port. However, once converted, reclaiming the pin for reprogramming requires both a **12V High-Voltage (HV) pulse** and a **hard power reset**. 

This repository contains the design files for an advanced hardware programmer that automates this entire recovery sequence cleanly. It features a triple-source input multiplexer designed to be entirely transparent to **Power Profilers (like the Nordic PPK2)**, allowing you to debug, flash, and accurately measure power consumption without swapping cables or losing serial terminal access.

---

## 🛠️ Key Architectural Features

### 1. Automated 12V HV Activation & Power Cycling
* **Automated Recovery:** Hardware automatically handles the delicate sequencing required to un-brick or re-flash chips using the UPDI pin for serial communication.
* **12V Boost Converter:** Integrates an `MT3608` high-frequency step-up boost converter providing a stable 12V programming rail directly from the USB bus.
* **Analog Switching:** Uses a `COS4561TR` analog multiplexer to safely inject the 12V pulse into the UPDI line only when the `RTS` control signal demands it.
* **Hard Power Reset:** Employs an `DOZ30P03` MOSFET setup controlled via the `DTR` line to physically break and cycle the target's power rail, satisfying the core condition for HV entry mode.

### 2. Triple-Input Power Architecture (Power Profiler Friendly)
To maintain accurate current metrics during low-power profiling, the board supports a versatile power-routing matrix utilizing a multi-position switch (`SW2`) and low R_DS(on) dual PMOS back-to-back isolation blocks (`DOZ30P03` pairs):
1. **Programming USB Power:** Power the Device Under Test (DUT) directly through the main flashing USB-C interface (regulated to 5V or 3.3V via the `AMS1117-3.3`).
2. **USB-C PD Pass-Through (Secondary Input):** A secondary USB-C port routes the `DP/DN` data lines and `CC1/CC2` lines straight to the DUT. This permits the target microcontroller to autonomously negotiate high-power profiles directly from an external PD charger while still being monitored.
3. **External Power Input (H1 Header):** A dedicated 2-pin isolated header allows you to hook up an external hardware power analyzer (such as the Power Profiler Kit II). This path bypasses the programmer's internal leakage paths, ensuring perfectly clean power data.

### 3. Core Processing & Connectivity
* **USB-to-UART Bridge:** Built around the robust `CH340X` IC, exposing high-baud rate serial terminals alongside hardware `RTS`/`DTR` control lines used for automated handshake sequencing.
* **Target Interface Pinout (`U6` 8-Pin Header):**
  1. `GND`
  2. `PIN_VCC` (Clean, selected rail)
  3. `PIN_CC2` (PD negotiation pass-through)
  4. `PIN_DP` (USB Data + pass-through)
  5. `PIN_DN` (USB Data - pass-through)
  6. `PIN_CC1` (PD negotiation pass-through)
  7. `PIN_UPDI` (HV / Serial shared line)
  8. `PIN_DTR` (Target Reset line)

---

## 🚀 The Programming & Debug Workflow

1. **Active Debugging:** The target runs its application code, utilizing its single UPDI pin as a `TX`/`RX` serial output. The `CH340X` reads this line natively for continuous data logging.
2. **Flash Command Issued:** Your compiler/toolchain signals the programmer to initiate a flash.
3. **HV Pulse & Power-Cycle Sequence:** The toolchain applies the 12V HV pulse via the analog switch to the `PIN_UPDI` line, followed immediately by dropping `DTR` to cycle the target's VCC power rail. Latching the HV pulse while executing this hard power reset forces the target microcontroller out of its shared serial/GPIO state and directly into UPDI programming mode.
4. **Firmware Upload:** The target receives the updated firmware payload over the established UPDI link and is instantly restored back to regular low-voltage operations or monitoring once the software releases the port.

---

## 🔌 Connection Setup Configurations

Depending on your project requirements, you can wire the programmer in either a minimal data-link configuration or a full pass-through configuration for advanced power negotiation.

### 📌 Setup 1: Minimal Connection (Power & UPDI Only)

This setup is ideal if you only need to flash and debug the ATtiny without utilizing any USB pass-through features. You can use a simple sliced USB cable or standard hookup wires directly to the target board.

#### 🗺️ Wiring Map
| Programmer Header Pin (`U6`) | Signal Name | Target ATtiny DUT Connection |
| :--- | :--- | :--- |
| **Pin 1** | `GND` | Ground (`GND`) |
| **Pin 2** | `PIN_VCC` | Power Input (`VCC` / `VBUS`) |
| **Pin 7** | `PIN_UPDI` | Repurposed UPDI Pin |

* **Leave Disconnected:** Pins 3, 4, 5, 6 and 8.

---

### ⚡ Setup 2: High-Power & PD Profiling Connection (Power over Second USB-C + UPDI)

Use this setup when the ATtiny DUT needs to interact with the second USB-C port to self-negotiate power profiles (USB Power Delivery) from an external charger, while maintaining the automated UPDI programming link.

#### 🗺️ Wiring Map
| Programmer Header Pin (`U6`) | Signal Name | USB-C Breakout / Target DUT Connection |
| :--- | :--- | :--- |
| **Pin 1** | `GND` | Ground (`GND`) |
| **Pin 2** | `PIN_VCC` | Negotiated `VBUS` Power |
| **Pin 3** | `PIN_CC2` | USB-C Configuration Channel 2 (`CC2`) |
| **Pin 4** | `PIN_DP` | USB Data Plus (`D+`) |
| **Pin 5** | `PIN_DN` | USB Data Minus (`D-`) |
| **Pin 6** | `PIN_CC1` | USB-C Configuration Channel 1 (`CC1`) |
| **Pin 7** | `PIN_UPDI` | Repurposed UPDI Pin |

* **How it routes:** The data lines (`DP`/`DN`) and configuration lines (`CC1`/`CC2`) pass straight through the programmer to the target. This lets the ATtiny talk directly to the power supply connected to the secondary USB-C port to request higher voltages/currents safely, while the UPDI line remains tied to the programming host.

If you enjoy this project and want to support its development, you can buy me a coffee:

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/W1L623I6WG)
