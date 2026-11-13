# STM32F429 Bootloader

A custom bootloader for the **STM32F429ZI** (Cortex-M4, 180 MHz) on the **STM32F429I-DISC1** Discovery board. It is written in C with the STM32 HAL and built with STM32CubeIDE.

The bootloader lives in the first 64 KB of flash. After start-up it hands control to a user application stored at `0x08040000`: it sets up the stack pointer, de-initializes the clock, HAL and SysTick, and jumps to the application's reset handler. Boot progress is logged over UART, and the user button can hold the board in the bootloader.

## Status

| Feature | State |
|---|---|
| Bootloader in its own flash region (custom linker script) | Done |
| Jump to application (MSP load, reset-handler call, peripheral/SysTick de-init) | Done |
| UART boot log (USART1, 115200 8N1, `printf` retargeted) | Done |
| LED status and user-button hold | Done |
| Firmware reception over UART | Planned |
| Flash erase / write from the bootloader | Planned |
| CRC check of the application image (CRC peripheral is already initialized) | Planned |
| Validity check of the application before jumping | Planned |

## Hardware

- STM32F429I-DISC1 board (STM32F429ZITx, 2 MB flash, 192 KB RAM)
- Clock: HSI 16 MHz, PLL M=8, N=180, P=2, giving a 180 MHz system clock (over-drive enabled)
- USART1 for the log output (115200 baud, 8N1)
- PG13: green LED (LD3), PA0: blue USER button
- ST-Link (on board) for flashing and debugging

## Flash memory layout

| Region | Address | Size | Content |
|---|---|---|---|
| Bootloader | `0x08000000` | 64 KB (sectors 0 to 4) | This project |
| Unused gap | `0x08010000` | 192 KB | Free |
| Application | `0x08040000` | up to 1792 KB (sectors 6 to 11) | User firmware |

The bootloader linker script (`STM32F429ZITX_FLASH.ld`) limits `FLASH` to `LENGTH = 64K`, so the build fails if the bootloader outgrows its region. `STM32F429ZITX_RAM.ld` is the usual CubeIDE script for debugging in RAM.

## Boot flow

```
Reset
  -> HAL_Init, SystemClock_Config (180 MHz)
  -> GPIO and USART1 init
  -> print banner, turn LED on, wait 2 s
  -> loop:
       USER button pressed?
         yes -> blink LED once per second while the button is held
         no  -> goto_application()
```

### How the jump works

```c
#define APP_START_ADDRESS 0x08040000

void (*app_reset_handler)(void) =
    (void*)(*((volatile uint32_t*)(APP_START_ADDRESS + 4U)));

HAL_RCC_DeInit();
HAL_DeInit();
__set_MSP(*(volatile uint32_t*)APP_START_ADDRESS);  /* app's initial stack pointer */
SysTick->CTRL = 0;  SysTick->LOAD = 0;  SysTick->VAL = 0;
app_reset_handler();                                /* jump */
```

The first word of the application's vector table is its initial stack pointer and the second word is its reset handler.

## Application requirements

An application started by this bootloader must be built for the offset address:

1. In its linker script, set `FLASH` to `ORIGIN = 0x08040000`.
2. In `system_stm32f4xx.c`, set the vector table offset to match:
   ```c
   #define USER_VECT_TAB_ADDRESS
   #define VECT_TAB_OFFSET  0x00040000U
   ```

## Build and flash

1. Open STM32CubeIDE and choose `File > Import > Existing Projects into Workspace`, then select this folder.
2. Build the project with the **Debug** configuration.
3. Flash the bootloader to the board with the ST-Link (`Run > Debug`, or STM32CubeProgrammer).
4. Flash your application to `0x08040000` (for example with STM32CubeProgrammer).
5. Open a serial terminal on the ST-Link virtual COM port at 115200 baud to see the boot log.

## Project structure

```
Core/                       Application code (main.c, HAL init, interrupts)
Drivers/                    STM32 HAL and CMSIS
STM32F429ZITX_FLASH.ld      Linker script: bootloader in the first 64 KB of flash
STM32F429ZITX_RAM.ld        Linker script: debug in RAM
Stm32f429_Bootloader.ioc    STM32CubeMX configuration
```
