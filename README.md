# MBS-BS5
Research notes into the MBS-BS5 BMS module made by [MPS](https://www.acermps.com) found in some Biltema e-bike batteries, firmware 19.0322.11.

My battery stopped working and so I decided to understand why. The top-most LED indicator (4) is blinking, indicating "malfunction".

## SWD
SWD is broken out to soldered headers, one per MCU. The one closest to the LED indicators seem to run the show. Neither MCU is read protected.

| Pin  | Function |
| - | - |
| 1 | VDD |
| 2 | PA14 (SWCLK) |
| 3 | VSS |
| 4 | PA13 (SWDIO) |
| 5 | BOOT0 |
| 6 | ? |
| 7 | PA9 (UART1_TX) |
| 8 | PA10 (UART1_RX |

## Main MCU (STM32F091)
### Connections
| Pin | Function |
| - | - |
| PB3 | Green LED4 |
| PB2 | Green LED3 |
| PB1 | Green LED2 |
| PB0 | Green LED1 |

### Debug UART (UART1)
* Debug UART needs modified firmware
* SRAM is initialized with some funky magic/compression (at FUN_08004a00) but the patch is 0x0801a23f needs to change from 0x01 to 0x00
* UART is running 38400 baud, 7N1

#### Example log
```
I/MAIN( 679): Application UP!fw: 19.0322.11 hw: 7
E/ML5236(  80): Error_State=3
```

### UART2
UART2 is being initialized, purpose unknown. My guess is communication with the second MCU or the charger.

### CAN
CAN is being initialized, purpose unknown. My guess is communication with the second MCU or the charger.

### SPI
SPI is setup to read/write from external flash memory. The onboard LAPIS ML5236 also communicates over SPI but unclear if this MCU, the secondary MCU or both is/are connected.
