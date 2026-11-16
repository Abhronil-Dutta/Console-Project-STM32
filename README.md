# STM32 Hardware-Accelerated Retro Console

A custom, bare-metal handheld game console engineered on the STM32F407G-DISC1 Cortex-M4 microcontroller. This project demonstrates advanced embedded systems programming by replacing standard blocking hardware abstraction layers (HAL) with a custom hardware-accelerated display driver utilizing Direct Memory Access (DMA) over SPI. 

The architecture currently runs a fully playable embedded Snake clone to validate the high-performance, non-blocking rendering pipeline and state-based graphical updates.

## Core Features

* **Hardware-Accelerated Rendering Pipeline:** Bypasses standard blocking HAL SPI libraries (`HAL_SPI_Transmit`) in favor of asynchronous DMA transfers (`HAL_SPI_Transmit_DMA`). This offloads pixel data routing entirely to the DMA controller.
* **Dirty Rectangle Updating Algorithm:** Achieves strict framerate targets by calculating state changes and transmitting localized sprite updates (e.g., character movement) rather than executing bandwidth-heavy full-frame buffer clears.
* **Asynchronous Game Engine Loop:** A non-blocking architecture that fully decouples the Cortex-M4 CPU from the SPI bus, allowing concurrent execution of edge-detected input polling, physics calculations, and background rendering.
* **Custom Driver Implementation:** Heavily modified ILI9341 display driver customized for STM32 hardware integration, specifically designed to handle DMA callback interrupts for precise Chip Select (CS) line management.

## System and Clock Configuration

To meet the high bandwidth demands of an RGB565 display without stalling the CPU, the microcontroller's clock tree and peripherals were manually tuned to their physical maximums:

* **System Clock (SYSCLK):** 168 MHz. Configured by routing the 16 MHz HSI through the Main PLL (M=8, N=168, P=2).
* **APB2 Bus Clock:** 84 MHz (SYSCLK / 2). Critical for maximizing the speed of the SPI1 peripheral.
* **SPI1 Protocol:** Transmit Only Master, 8-Bit Data Width, CPOL Low, CPHA 1 Edge.
* **SPI1 Baud Rate:** 10.5 Mbit/s (Prescaler of 8). *Engineering Note: While the APB2 bus supports a theoretical SPI speed of 42 Mbit/s, the prescaler is intentionally lowered to 10.5 Mbit/s to maintain signal integrity and prevent capacitive rounding across physical breadboard jumper wires.*
* **GPIO Slew Rates:** All SPI and control pins are configured to 'Very High' output speeds to prevent square-wave degradation during high-frequency transmission.

## Direct Memory Access (DMA) Architecture

Implementing DMA required solving specific hardware-level synchronization constraints:

* **Stream Configuration:** DMA2 Stream 3 is configured for Memory-to-Peripheral transfer, triggered by the `SPI1_TX` request.
* **Data Width Alignment:** Both Peripheral and Memory data widths are strictly set to **Byte**. This exact matching with the 8-bit SPI peripheral prevents silent hardware transfer faults. 16-bit color buffers are pre-formatted via byte-swapping in software prior to transmission.
* **Initialization Sequence:** To prevent silent stream failures, `MX_DMA_Init()` is strictly sequenced prior to `MX_SPI1_Init()` in the bootloader, ensuring the DMA controller is fully powered and listening before the SPI peripheral attempts to bind to it.
* **Interrupt Management:** DMA Tx Complete interrupt is enabled via the NVIC. 

## Pin Mapping (STM32F407G-DISC1)

| Peripheral | Pin | Mode | Description | 
| ----- | ----- | ----- | ----- | 
| **ILI9341 Screen** | `PA5` | GPIO_Alternate | SPI1_SCK (Serial Clock) | 
| **ILI9341 Screen** | `PA7` | GPIO_Alternate | SPI1_MOSI (Master Out Slave In) | 
| **ILI9341 Screen** | `PC1` | GPIO_Output | Chip Select (CS) - Handled via DMA IRQ Callback | 
| **ILI9341 Screen** | `PC2` | GPIO_Output | Reset (RST) | 
| **ILI9341 Screen** | `PC3` | GPIO_Output | Data/Command (DC) | 
| **Hardware Input** | `PA1` | GPIO_Input | Button 1 (Pull-up) | 
| **Hardware Input** | `PA2` | GPIO_Input | Button 2 (Pull-up) | 

## Software Architecture Overview

### The Asynchronous Rendering Pipeline

Standard embedded graphics pipelines stall the CPU during transmission, creating a bottleneck linked to the SPI bus speed. This engine decouples the CPU from the display. When a sprite update is required, the driver sets the display's internal SRAM address window, raises the Data/Command pin, and triggers the DMA controller:

```c
// Halt if a previous frame is still transmitting
while(spi_dma_busy) {}

// Lock the pipeline and fire the DMA stream
spi_dma_busy = 1; 
HAL_SPI_Transmit_DMA(&ILI9341_SPI_PORT, (uint8_t*)data, w * h * 2);
```

Following this call, the main `while(1)` game loop instantly resumes execution. Once the final byte of the sprite buffer exits the STM32's hardware shift register, a hardware interrupt fires `HAL_SPI_TxCpltCallback()` to pull the Chip Select line HIGH, formally ending the SPI transaction and unlocking the software bus for the next frame.

## Build Instructions

1. Clone this repository to your local machine.
2. Import the project into **STM32CubeIDE**.
3. Compile using the `Release` build profile for optimal performance.
4. Flash to an STM32F407-based board using the integrated ST-LINK.
5. Wire the ILI9341 SPI display and input buttons according to the pinout table above.

## License

This project is open-source and available under the MIT License.