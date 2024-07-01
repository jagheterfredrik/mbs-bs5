# MBS-BS5
Research notes into the MBS-BS5 BMS module found in Biltema bikes, firmware 19.0322.11.

My e-bike battery stopped working and so I decided to understand why. The top-most LED indicator (4) is blinking, indicating "malfunction".

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
| 7 | VDDIO2 |
| 8 | PA9 (UART1_TX) |

## Debug UART
* Debug UART needs modified firmware
* SRAM is initialized with some funky magic/compression (at FUN_08004a00) but the patch is 0x0801a23f needs to change from 0x01 to 0x00
* 38400 baud, 7 data bits, 1 stop bit.

### Example log
```
I/MAIN( 679): Application UP!fw: 19.0322.11 hw: 7
E/ML5236(  80): Error_State=3
```
