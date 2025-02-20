# MBS-BS5
Research notes into the MBS-BS5 BMS module made by [MPS](https://www.acermps.com) found in some Biltema e-bike batteries, main MCU firmware 19.0322.11. After blowing up my initial module (v2.2), I got spares from a local ebike repair shop (shout out [Batteridoktorn](https://batteridoktorn.se/)), the new modules are v2.1 and v2.0. The main MCU firmware on the diagnosed 2.1 is 18.1029.91.

My battery stopped working and so I decided to understand why. The top-most LED indicator (4) is blinking, indicating "malfunction". The reason behind the malfunction code is a blown fuse. The reason for the fuse being blown is in question, however as [the fuse](https://www.eaton.com/content/dam/eaton/products/electronic-components/resources/data-sheet/eaton-scf9550-self-control-fuse-data-sheet-elx1135-en.pdf) is software-controlled.

## SWD
SWD is broken out to soldered headers, one per MCU. The one closest to the LED indicators seem to run the show. Neither MCU is read protected.

| Pin  | Function |
| - | - |
| 1 | VDD (3.3V) |
| 2 | PA14 (SWCLK) |
| 3 | VSS (GND) |
| 4 | PA13 (SWDIO) |
| 5 | nRST |
| 6 | ? |
| 7 | PA9 (UART1_TX) |
| 8 | PA10 (UART1_RX |

## Main MCU (STM32F091)
### Connections
| Pin/peripheral | Function |
| - | - |
| PB3 | Green LED4 |
| PB2 | Green LED3 |
| PB1 | Green LED2 |
| PB0 | Green LED1 |
| PB12 | External flash CS |
| SPI2 | External flash SPI |

### Debug UART (UART1)
* Debug UART needs modified firmware
* SRAM is initialized with some funky magic/compression (at FUN_08004a00) but the patch is 0x0801a23f needs to change from 0x01 to 0x00
* UART is running 38400 baud, 7N1

#### Example log
```
I/MAIN( 679): Application UP!fw: 19.0322.11 hw: 7
E/ML5236(  80): Error_State=3
```

### Config parameters
At 0x0801D000 there's a struct of configuration parameters written during final assembly. The full list of things can be reversed from the mBike app. I'll list some of them:

- 0x00 - uint32_t n_cells
- 0x04 - uint32_t pack_version
- 0x08 - uint32_t max_capacity
- 0x0c - uint32_t full_cell_voltage
- 0x36 - uint8_t ble_enabled
- 0x100 uint32_t battery_id
- 0x108 - uint16_t crc16 of the parameters

### UART2
UART2 is being initialized, used to communication with the second MCU.

### UART3
UART3 is initialized if ble_enabled is set in the configuration parameters.

### CAN
CAN is being initialized, used to communicate with the charger.

### SPI
SPI is setup to read/write from external flash memory. The onboard LAPIS ML5236 also communicates over SPI but unclear if this MCU, the secondary MCU or both is/are connected.

### Firmware
At 0x0801d800 there's a string containing serial numbers, hardware id and manufacturing date. Interestingly the manufacturing date is parsed from this string, parsed into a unix timestamp and stored in RAM. The reason to parse it from the string is probably to have static firmware file that's customized at the production line. The reason for parsing it at all is however kinda sus.

## Secondary MCU (STM32F071)
Communicates with Main MCU over 9600 baud USART2 using the AMPS protocol.

Has a much smaller firmware that the main MCU.

## AMPS Protocol
A proprietary TLV-style protocol encoded as

```
"AMPS" [uint32_t length] [uint16_t cmd] [data] [crc16]
```
Length is packet length excluding CRC. Both length and cmd is little endian.
The CRC excludes the static AMPS packet header.

### Commands
#### From Main MCU
 - 0x03 - Command response
   - data[0] - uint16_t code:
     - 0x00 - ACK?
     - 0x01 - error crc check fail
     - 0x02 - error unknown cmd
     - 0x11 - error buffer overrun?
 - 0x05
 - 0x07
   - data[0] - int32 workstate
   - data[4] - int8 temperature 1 (0.1 degrees resolution)
   - data[6] - int8 State of Charge
   - data[7] - int8 State of Health
 - 0x09
   - data[0] - float voltage (in mV)
   - data[4] - float current (in mA)
 - 0x0F - MCU date/time response
 - 0x17 - 0x72 bytes, includes firmware version
   - data[106] - char[10] firmware version
 - 0x19
 - 0x1A - No data, len 0
 - 0x1D
 - 0x20
 - 0x22
   - Sent as response to 0x21 from secondary. Response includes 0x78 bytes of configuration data.
 - 0x101
 - 0x103
 - 0x104
 - 0x106

#### From Secondary MCU
 - 0x21 - No data, len 0, request main MCU to send over parameters, main responds with 0x22

## App
The main MCU has support for Bluetooth, connected to USART3. To utilize it, there's [an app](https://play.google.com/store/apps/details?id=mps.mps_bike), which probably talks the AMPS protocol. The app references things that indicate upgrade functionality, e.g. link to [firmware bins]([url](https://mpspackfw.firebaseapp.com/)).
