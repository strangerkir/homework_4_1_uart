# UART: STM32 ↔ ESP32

An educational two-firmware project that connects an STM32F411 and an ESP32-S3 over UART. A button on either board sends an `ON` or `OFF` command to control the other board's LED.

## Features

- STM32 button input alternates `ON`/`OFF` commands for the ESP32 LED.
- ESP32 button input alternates `ON`/`OFF` commands for the STM32 LED.
- Both sides use newline-terminated UART text commands at 9600 baud, 8 data bits, no parity, and 1 stop bit (8N1).
- The STM32 acknowledges commands it receives with `ACK ON` or `ACK OFF`. The ESP32 checks those acknowledgements and retries up to three times with a 200 ms timeout.
- STM32 diagnostic messages are sent separately over USART1 at 115200 baud.

## Hardware and wiring

Cross-connect the UART signals and share ground:

| ESP32-S3 | STM32F411 | Purpose |
| --- | --- | --- |
| GPIO17 (TX) | PA3 (USART2_RX) | ESP32 to STM32 |
| GPIO18 (RX) | PA2 (USART2_TX) | STM32 to ESP32 |
| GND | GND | Common signal reference |

Use compatible 3.3 V UART logic; do not connect the UART pins to 5 V signals. The listed GPIOs must be available on the specific boards. Connect an LED and suitable current-limiting resistor to ESP32 GPIO4 if the board does not provide an LED on that pin. STM32 PC13 is configured as an active-low LED output; whether it drives an onboard LED depends on the board.

Both button inputs use pull-ups and are active low (pressed connects the pin to ground): STM32 PA0 and ESP32 GPIO0. GPIO0 is an ESP32-S3 strapping pin, so holding its button during reset can affect boot mode. The ESP32 firmware uses GPIO0 as a button; verify that this matches the board's button wiring.

### Pin assignments

| Board | Signal | Pin | Notes |
| --- | --- | --- | --- |
| STM32F411 | UART link TX / RX | PA2 / PA3 (USART2) | 9600 baud, 8N1 |
| STM32F411 | UART log TX / RX | PA9 / PA10 (USART1) | 115200 baud; logs are transmitted on TX (PA9) |
| STM32F411 | Button | PA0 | Active low, pull-up enabled |
| STM32F411 | LED | PC13 | Active low |
| ESP32-S3 | UART link TX / RX | GPIO17 / GPIO18 (UART1) | 9600 baud, 8N1 |
| ESP32-S3 | Button | GPIO0 | Active low, pull-up enabled |
| ESP32-S3 | LED | GPIO4 | Firmware drives high for ON and low for OFF |

## Build and flash

### ESP32-S3

Install and activate ESP-IDF, then run these commands from the `esp32/` project directory. Replace `/dev/ttyUSB0` with the serial port for your board.

```sh
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### STM32F411

1. Open the `stm32/` project in STM32CubeIDE as an existing project and allow it to resolve or regenerate the CubeMX configuration from `stm32.ioc`.
2. Install the STM32Cube F4 firmware package version 1.28.3 if it is not already available.
3. Build the Debug or Release configuration, then program the board using its supported programmer (for example, an attached ST-LINK).

The current STM32CubeIDE `.cproject` contains absolute HAL/CMSIS include paths under `/home/gfnow/STM32Cube/Repository/`. On another machine, update those include paths in the project settings or regenerate them through STM32CubeIDE so they point to that machine's STM32Cube F4 1.28.3 installation.

## Operation and limitations

- Pressing the STM32 PA0 button sends alternating `ON` and `OFF` commands to the ESP32. The STM32 firmware does not wait for an acknowledgement; while the button remains held, it sends another command after each 100 ms debounce delay, so release the button promptly for a single toggle.
- Pressing the ESP32 GPIO0 button sends an alternating command to the STM32 and waits for the matching acknowledgement. It retries up to three times if an acknowledgement is not received.
- Commands are plain text lines terminated by `\n`; `\r` is ignored. This is a simple demonstration protocol, not reliable state synchronization: there is no shared state recovery if a command or acknowledgement is lost.
- To view STM32 logs, connect a 3.3 V-compatible USB-to-UART adapter to PA9 (adapter RX), PA10 (adapter TX, if needed), and GND, and set the terminal to 115200 baud. The firmware transmits logs on PA9.
