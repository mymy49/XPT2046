# XPT2046

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Doxygen Build](https://github.com/mymy49/XPT2046/actions/workflows/doxygen.yml/badge.svg)](https://github.com/mymy49/XPT2046/actions)
[![Doxygen API Docs](https://img.shields.io/badge/docs-API%20Reference-brightgreen.svg)](https://mymy49.github.io/XPT2046/)

High-performance, event-driven C++ driver for the **XPT2046** 4-wire resistive touch screen controller, built for the **yss OS** embedded framework.

---

## 1. Overview & Key Features

The **XPT2046** is a widely used 4-wire resistive touch screen controller featuring a 12-bit, 125 kHz successive approximation register (SAR) Analog-to-Digital Converter (ADC). This module integrates seamlessly into **yss OS** by utilizing its Hardware Abstraction Layer (HAL) and real-time threading primitives.

### Key Highlights
- **Hardware Integration & Control**:
  - Full support for XPT2046 command structures via standard SPI (Mode 0, up to 2 MHz clock rate, 8-bit transfer).
  - 12-bit coordinate acquisition on both X and Y channels with configurable low-power modes.
- **Asynchronous, Thread-Safe Architecture**:
  - Inherits from `Trigger` (`yss/Trigger.h`), executing touch acquisition and event generation within an independent, lightweight OS thread (512-byte stack).
  - Driven by the active-low `PENIRQ` hardware interrupt (falling edge), eliminating the overhead and latency of periodic CPU polling.
- **Pointer Event Pipeline**:
  - Inherits from `ResistiveTouch` (`hal/ResistiveTouch.h`), providing raw ADC to screen coordinate calibration (`calculateX()`, `calculateY()`).
  - Emits standard touch events: **Touch Down** (`pushDownEvent()`), **Continuous Drag** (`pushDragEvent()`), and **Touch Up** (`pushUpEvent()`).
- **Contact Stabilization & Multitasking**:
  - Integrated 30 ms contact debounce delay to filter mechanical bounce and false triggers.
  - Cooperative multi-threaded yield (`thread::yield()`) during sustained drags to ensure real-time responsiveness across the entire system.

---

## 2. System Specifications & Hardware Environment

### Supported Targets & Tools
- **Target Microcontroller / Development Board**:
  - Nuvoton NuMicro® M251 series (e.g., **NuMaker-M251KG** / Cortex®-M23).
  - Any microcontroller platform supported by the **yss OS** framework.
- **Shield & Display Module**:
  - Designed for Arduino UNO-compatible touch LCD shields, such as the **2.8-inch TFT Touch Shield Rev2.1** (featuring ST7789V LCD + XPT2046 Touch Controller).
- **Required Software & Toolchains**:
  - Toolchain: ARM GNU Toolchain or CrossWorks for ARM (v5.x or higher).
    - **Note on CrossWorks for ARM**: CrossWorks for ARM does not natively support the Nuvoton M251 series. You must install the custom [CrossworksForARM_Nuvoton_M251 Support Package](https://github.com/mymy49/CrossworksForARM_Nuvoton_M251) before building M251 projects.
  - Framework: [yss OS](https://github.com/mymy49/yss) (embedded RTOS & HAL).
  - Language: C++11 or higher.

### Pin Map & Hardware Connections
The table below illustrates the hardware pin assignments between the **NuMaker-M251KG** board and the **XPT2046** touch controller on the 2.8-inch TFT Touch Shield Rev2.1:

| Signal Name | XPT2046 Pin | NuMaker-M251KG Pin | MCU Peripheral / Function | Description |
| :--- | :--- | :--- | :--- | :--- |
| **MOSI** | DIN | `PA.0` | `Gpio::PA0_SPI0_MOSI` | SPI Master Output / Slave Input |
| **MISO** | DOUT | `PA.1` | `Gpio::PA1_SPI0_MISO` | SPI Master Input / Slave Output |
| **CLK** | DCLK | `PA.2` | `Gpio::PA2_SPI0_CLK` | SPI Serial Clock |
| **CS** | /CS (Active Low) | `PC.3` | GPIO Output (Active Low) | Touch Controller Chip Select |
| **PENIRQ** | /PENIRQ (Active Low) | `PC.5` | GPIO External Interrupt | Touch Interrupt Pin (Falling Edge) |
| **BUSY** | BUSY | `NC` | Unused (`{0, 0}`) | Optional busy status indicator |

---

## 3. Installation & Project Setup

### Cloning with Submodules
To clone a project that includes this repository as a submodule:
```bash
git clone --recursive https://github.com/mymy49/NuMaker-M251KG_example.git
```

To add this repository as a submodule into your existing yss OS application:
```bash
git submodule add https://github.com/mymy49/XPT2046.git module/XPT2046
```

### Prerequisites: Nuvoton M251 Support Package for CrossWorks for ARM
CrossWorks for ARM does not natively provide device support for Nuvoton NuMicro® M251 microcontrollers in its default package manager. A custom device support package must be installed beforehand:

1. Clone or download the [CrossworksForARM_Nuvoton_M251](https://github.com/mymy49/CrossworksForARM_Nuvoton_M251) repository.
2. Open the package project file (`.hzp`) in CrossWorks for ARM and build it (`F7` or `Build -> Build Project`) to generate the package file (`.hzq`).
3. In CrossWorks for ARM, navigate to **Tools -> Packages -> Manually Install Packages...** and select the generated `.hzq` package file.
4. Once installed, M251 target device definitions, CMSIS headers, and project templates will be fully available in CrossWorks for ARM.

### Build and Firmware Flashing
- **CrossWorks for ARM**:
  1. Ensure the [CrossworksForARM_Nuvoton_M251](https://github.com/mymy49/CrossworksForARM_Nuvoton_M251) package has been installed.
  2. Open the project solution file (e.g., `Example.hzp`).
  3. Select target configuration (`Example THUMB Release` or `Example THUMB Debug`).
  4. Build solution via `F7` or `Build -> Build Solution`.
  5. Connect the board via Nu-Link ICE and flash with `Ctrl+D` (`Target -> Download`).
- **Command Line (pyOCD / OpenOCD)**:
  ```bash
  # Flash prebuilt binary using pyocd
  pyocd flash -t m251kg firmware.bin
  ```

---

## 4. Quick Start & Example Code

Below is a complete, real-world example demonstrating how to initialize the SPI peripheral, configure the `XPT2046` instance, and integrate it with the `TouchEventDispatcher` under yss OS:

### 1. Hardware Initialization (`bsp.cpp`)
```cpp
#include <bsp.h>
#include <yss/instance.h>
#include <module/XPT2046/XPT2046.h>

// Global touch controller and event dispatcher instances
XPT2046 touch;
TouchEventDispatcher dispatcher(20, 2048);

void initializeBoard(void)
{
    // 1. Configure SPI0 GPIO alternate functions
    gpioA.setAsAltFunc(0, Gpio::PA0_SPI0_MOSI);
    gpioA.setAsAltFunc(1, Gpio::PA1_SPI0_MISO);
    gpioA.setAsAltFunc(2, Gpio::PA2_SPI0_CLK);

    // 2. Initialize SPI0 peripheral
    Spi::config_t spi0Config = 
    {
        Spi::MODE_MAIN
    };

    spi0.enableClock();
    spi0.initialize(spi0Config);
    spi0.enableInterrupt();

    // 3. Configure Touch Control GPIOs (Chip Select & Interrupt)
    gpioC.setAsOutput(3); // CS
    gpioC.setAsOutput(5); // PENIRQ

    // 4. Initialize XPT2046 Driver
    XPT2046::config_t touchConfig = 
    {
        spi0,           // Spi &spi;
        {&gpioC, 3},    // pin_t chipSelect;
        {0, 0},         // pin_t busy; (unused)
        {&gpioC, 5},    // pin_t isr;  (PENIRQ)
    };

    touch.initialize(touchConfig);

    // 5. Register touch driver as the pointer device for dispatcher
    dispatcher.setPointerDevice(touch);
}
```

### 2. Handling Touch Events in Application Code
```cpp
#include <yss.h>
#include <bsp.h>

void touchEventCallback(PointerDevice::event_t event)
{
    switch(event.event)
    {
    case PointerDevice::EVENT_TOUCH_DOWN:
        // Touch pressed at (event.x, event.y)
        break;

    case PointerDevice::EVENT_TOUCH_DRAG:
        // Touch dragged to (event.x, event.y)
        break;

    case PointerDevice::EVENT_TOUCH_UP:
        // Touch released
        break;
    }
}
```

### 3. Coordinate Calibration
Because `XPT2046` inherits from `ResistiveTouch`, screen coordinates are mapped using calibration coefficients:
```cpp
// Set calibrated display dimension and touch bounds
touch.setCalibration(displayWidth, displayHeight, p1Raw, p1Screen, p2Raw, p2Screen);

// Retrieve raw 12-bit ADC coordinates when building custom calibration wizards:
uint16_t rawX = touch.getLastConvertedAdcX();
uint16_t rawY = touch.getLastConvertedAdcY();
```

---

## 5. Documentation & References

- **Doxygen API Reference**: [https://mymy49.github.io/XPT2046/](https://mymy49.github.io/XPT2046/)
- **XPT2046 Datasheet**: [XPTEK XPT2046 4-Wire Resistive Touch Screen Controller Specification](http://www.xpttek.com/)
- **Doxygen Main Page**: This `README.md` file serves as the main page of the generated Doxygen documentation (`USE_MDFILE_AS_MAINPAGE = README.md`).

---

## 6. License & Author

- **Author**: Yoon-Ki Hong
- **License**: MIT License. See [LICENSE](LICENSE) for full terms and conditions.
