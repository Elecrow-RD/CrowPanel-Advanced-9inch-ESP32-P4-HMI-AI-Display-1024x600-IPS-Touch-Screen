# ESP32-P4 9.0-Inch Product Hardware Driver Manual

| Item | Details |
|---|---|
| Document Version | V1.0 |
| Applicable Hardware | ESP32-P4 Display 9.0 inch V1.2 |
| Schematic Version | V1.2, 2026-05-05 |
| Document Date | 2026-07-30 |
| Author | OpenAI Codex (compiled from the project schematics and verified sample code) |
| Scope | Hardware maintenance, ESP-IDF driver porting, production issue troubleshooting, and onboarding handoff |

## 1. Document Purpose and Evaluation Criteria

This document cross-validates `1.2/ESP32-P4 Display 9.0 inch V1.2.sch`, the PDF/PCB files of the same version, and the tested examples in `idf-code/`. Conclusions are determined according to the following priority:

1. Actual configurations in successfully tested driver code;
2. Network connections in the EAGLE `.sch` schematic;
3. Text in PDF drawings, README files, and Kconfig default values;
4. General device datasheet knowledge is used only for explanation and does not constitute verified evidence for this board.

Status definitions:

- **A - Code and drawings are consistent**: Can be used directly as the baseline for the current board support package.
- **B - Code takes precedence/discrepancy exists**: Use the successfully tested code and retain a record of the discrepancy.
- **C - Present in drawings, but no functional driver in this project**: The hardware has been identified, but software verification cannot be claimed.
- **D - Optional external module**: The example has verified the interface driver, but the device may not be installed on the mainboard.

> Note: The repository does not contain a unified `sdkconfig`, build artifacts, test logs, or BOM. In this document, “verified” refers to successfully tested configurations represented by the course/sample code provided with the project. It does not mean that the physical hardware was reconnected and electrically tested specifically for this document.

## 2. Product Architecture Overview

The main controller is the ESP32-P4NRW32. The ESP32-P4 itself does not integrate 2.4 GHz Wi-Fi. The board includes a separate ESP32-C6-MINI-1-N4, which provides Wi-Fi through an SDIO link using `esp_hosted`/`esp_wifi_remote`. The primary display is a 1024×600 dual-data-lane MIPI-DSI panel, with a GT911 I2C touch controller. The board also includes MicroSD, USB, a PDM microphone, I2S digital amplifier/audio codec circuitry, a MIPI-CSI camera interface, UART and wireless expansion interfaces, battery charging, and multiple power rails.

## 3. Peripheral Overview

| Category | Peripheral/Device | Primary Interface | Key Pins/Networks | Software Evidence | Status |
|---|---|---|---|---|---|
| Main Controller | ESP32-P4NRW32 (U7) | SoC | All board GPIO, MIPI, USB, SDMMC | All ESP-IDF examples | A |
| Wireless Coprocessor | ESP32-C6-MINI-1-N4 (IC1) | SDIO 4-bit | P4 GPIO14–19; P4 GPIO32 EN | Lesson16/17 `esp_hosted` | A |
| External Flash | W25Q128JVSIQ (IC11) | Quad SPI | Dedicated FLASH_CS/CK/D/Q/WP/HOLD | Boot path; not directly configured by the application | C |
| Display | EK79007-compatible 9-inch LCD | MIPI-DSI 2-lane | DSI D0/D1/CLK; GPIO31 BL_EN | Lesson07/09/13/16 | A |
| Touch | GT911 (display assembly) | I2C0 + GPIO | SDA45, SCL46, RST40, INT42 | Lesson05/06/09/16 | A |
| Backlight | MT9201 + LED boost path | PWM/GPIO | GPIO31→LCD_BK_EN; GPIO29→power switch | Lesson07/09/10/13–15 | A/C |
| Storage | MicroSD (J5) | SDMMC 1-bit | CLK43, CMD44, D0=39 | Lesson08/12 | A |
| USB Device | USB2 Type-C (J16) | USB 2.0 FS/HS PHY | P4 DP/DM→USB2_D+/D- | Lesson06 TinyUSB HID | A |
| Download UART | CH340K (U1) + Type-C (J1) | UART0/USB-UART | GPIO37 TXD0, GPIO38 RXD0 | Boot/download hardware; no BSP | C |
| Expansion UART | J2 | UART1 | GPIO47 TX, GPIO48 RX, 3.3 V | Lesson04 uses UART2 peripheral mapping | A/B |
| Input UART | J10 | UART3 | GPIO34 TX, GPIO33 RX, 5 V/GND | Macros only; not initialized | C |
| PDM Microphone | LMA3526B381 (U176) | PDM/I2S0 RX | CLK24, DATA26; GPIO20 L/R | Lesson11 | A |
| Digital Amplifier/Speaker | NS4168 ×2, NS4263B | I2S1 TX + GPIO | LRCK21, BCLK22, DATA23, CTRL30 | Lesson11/12 | A |
| Audio ADC | ES7210 (IC5) | I2C + I2S/TDM | I2C1, MCLK/SCLK/LRCK/SDOUT | No register driver | C |
| Audio Codec | ES8311 (IC6) | I2C + I2S | I2C1, MCLK/SCLK/LRCK/DSDIN | No register driver | C |
| Camera | SC2336 (external FPC3) | MIPI-CSI 2-lane + SCCB | SDA12, SCL13, CSI D0/D1/CLK | Lesson13 | A/D |
| Temperature/Humidity | DHT20 (external J13) | I2C0 | SDA45, SCL46, address 0x38 | Lesson10 | D |
| LoRa | SX1262 (expansion module) | SPI3 + GPIO IRQ | SCK8/MISO7/MOSI6/BUSY9/CS10/IRQ27/RST28 | Lesson14 | D/B |
| 2.4G RF | nRF24 (expansion module) | SPI3 + GPIO IRQ | SCK8/MISO7/MOSI6/IRQ9/CE27/CS28 | Lesson15 | D/B |
| Expansion GPIO | J7 2×12 | GPIO/power | GPIO2–5, 25, 49–54; 3.3/5 V | No unified BSP | C |
| Peripheral I2C | J13 | I2C0 | GPIO45/46, 3.3 V, GND | Reusable from Lesson10 | A/D |
| Buttons | BOOT (K3), RESET (K4) | Hardware input | SPI_BOOT, P4_RST_EN; grounded when pressed | Boot/reset hardware | C |
| Battery/Charging | TP4059 (U2), J3 | Analog power | VBAT, CHG, STD | Managed by STC8; no P4 driver | C |
| Power Management MCU | STC8H1K08 (U14) | ADC/GPIO | ADC_VBAT, CHG/STD, red/green LEDs | No source code/firmware | C |
| Power Tree | MT3406/TLV62569, ME6211, MT3608, MT9201 | DC/DC/LDO | 5 V, 3.3 V, 1.2 V, 1.8 V, 2.8 V, LCD bias | Hardware auto-start/enable | C |

## 4. ESP32-P4 Main Controller and Core Resources

### 4.1 Main Controller and Clocks

- Device: U7 `ESP32-P4NRW32`; the main crystal Y2 is 40 MHz.
- Software layer: ESP-IDF; examples generally require IDF 5.4.x. FreeRTOS is provided by ESP-IDF.
- External boot Flash: IC11 `W25Q128JVSIQ`, connected to the dedicated `FLASH_CS/CK/D/Q/WP/HOLD` signals. These pins must not be assigned as general-purpose GPIO.
- A 32.768 kHz crystal, Y4, appears in the drawings, but the source name in its component library does not establish what it actually serves. No application code was found that directly configures it. The ownership and source of the low-speed clock must be separately confirmed in the production `sdkconfig` and on the physical board.

### 4.2 Assigned GPIO Summary (Code Takes Precedence)

| GPIO | Actual Function | Multiplexing/Conflict Notes |
|---:|---|---|
| 6/7/8 | Wireless expansion SPI3 MOSI/MISO/SCK | Corresponds to the J9 SPI2-labeled path; initialize only one RadioLib bus instance at a time |
| 9/10 | Wireless module IRQ/BUSY and CS | SX1262: BUSY9/CS10; nRF24: IRQ9 |
| 12/13 | Camera SCCB SDA/SCL | Level-shifted to CSI_SDA/SCL in the drawings |
| 14–19 | ESP32-C6 SDIO D3/D2/D1/D0/CLK/CMD | Dedicated to hosted Wi-Fi; do not use as general-purpose GPIO |
| 20 | PDM microphone L/R selection | Not configured in code; preserve the fixed schematic network |
| 21/22/23 | I2S1 LRCLK/BCLK/DOUT | Digital amplifier playback |
| 24/26 | PDM CLK/DIN | I2S0 microphone recording |
| 27/28 | Wireless IRQ/RESET or CE/CS; also UART passthrough TX/RX | Severe mutual exclusion; see the Risks section |
| 29 | LCD backlight power switch `LCD_BK_POWER` | Not controlled by the current display code |
| 30 | Audio amplifier control, active-low | Must not be used as a general-purpose LED/GPIO |
| 31 | LCD backlight PWM/enable | LEDC 30 kHz |
| 32 | ESP32-C6 EN | Wi-Fi coprocessor reset/enable |
| 33/34 | UART3 RX/TX (J10) | J10 provides 5 V power; this does not mean the signals are 5 V tolerant |
| 37/38 | UART0 TX/RX | CH340K download/logging |
| 39/43/44 | SDMMC D0/CLK/CMD | MicroSD 1-bit |
| 40/42 | GT911 RST/INT | Dedicated to touch |
| 41 | LCD_RESET | Current panel driver sets `reset_gpio_num=-1` and does not use it directly |
| 45/46 | I2C0 SDA/SCL | Shared by GT911, DHT20, and J13; the drawings include level shifting/pull-up networks |
| 47/48 | Expansion UART TX/RX | GPIO48 is incorrectly used by the LED example; simultaneous use is prohibited |
| 49–54, 2–5, 25 | J7 expansion GPIO | Verify the specific expansion board and voltage before use |

## 5. Display and Touch

### 5.1 LCD and MIPI-DSI

**Hardware Connections**

- LCD FPC: J21; DSI differential pairs are `DSI_DATA0_P/N`, `DSI_DATA1_P/N`, and `DSI_CLK_P/N`.
- Panel control: `LCD_RESET` corresponds to GPIO41; backlight enable `LCD_BK_EN` corresponds to GPIO31.
- Panel bias rails: `LCD_AVDD_9V6`, `LCD_VGH_18V`, `LCD_VGL_-6V`, `LCD_VCOM_3V3`, and `LCD_VDD`, generated by dedicated boost/LDO circuitry.

**Verified Software Configuration**

- Drivers: `espressif/esp_lcd_ek79007` 1.0.2, ESP-IDF MIPI-DSI/DPI drivers, LVGL 8.3.11, and `esp_lvgl_port` 2.6.0.
- Resolution/format: 1024×600, RGB565, 16 bpp.
- DSI: bus 0, 2 data lanes, lane rate 900 Mbps, and 8-bit DBI commands and parameters.
- DPI pixel clock: 51 MHz.
- Horizontal timing: back porch 160, sync 70, front porch 160.
- Vertical timing: back porch 23, sync 10, front porch 12.
- `reset_gpio_num=-1`; the driver resets the panel through panel commands and does not directly drive GPIO41.
- Key source: `idf-code/Lesson07-Turn_on_the_screen/peripheral/bsp_illuminate/bsp_illuminate.c`.

```c
esp_lcd_dsi_bus_config_t bus = {
    .bus_id = 0, .num_data_lanes = 2, .lane_bit_rate_mbps = 900,
};
// DPI: 51 MHz, 1024x600, RGB565
```

### 5.2 Backlight

- GPIO31 is configured as a standard push-pull output and then connected to LEDC: low-speed mode, timer0/channel0, PLL-divided clock, 30 kHz, 11-bit duty resolution, and no interrupt.
- Brightness values from 1–100 map to duty `brightness * 18 + 200`; 0 maps to 0. The caller must constrain the value to 0–100; otherwise, it may exceed the 11-bit maximum of 2047.
- The drawings also include GPIO29 `LCD_BK_POWER`, which controls the backlight input-power MOSFET. The current display BSP controls only GPIO31 and does not manage GPIO29. When porting low-power display shutdown, verify both the polarity and the power-on default state of GPIO29.

### 5.3 GT911 Capacitive Touch

- Bus: I2C0, SDA GPIO45, SCL GPIO46; the preferred 7-bit address is `0x5D`, with a fallback to `0x14` upon failure, as defined by the GT911 component macros.
- Speed: 400 kHz; internal pull-ups enabled, glitch filter count=7.
- Control: RST GPIO40, INT GPIO42; the driver configures reset=0 and interrupt=0, with INT managed by the GT911 component.
- Coordinate range: 1024×600; software polls using `esp_lcd_touch_read_data()`. The code does not install a GPIO42 ISR.
- Electrical connections: FPC2 in the drawings provides SCL/SDA/INT/RST/3.3 V/GND. The I2C network includes level shifting and external pull-ups and must not rely solely on the weak internal pull-ups.
- Software: `esp_lcd_touch_gt911` 1.1.3 + ESP-IDF `i2c_master` + LVGL input device.

## 6. Storage

### 6.1 MicroSD

- Interface: J5, SDMMC slot 0, 1-bit mode.
- Pins: CLK GPIO43, CMD GPIO44, D0 GPIO39. The drawings also include DATA1/DATA2/CS, but the current code does not use them.
- Clock: maximum 10 MHz; the internal pull-up flag is enabled, but hardware pull-ups on CMD/D0 should still be retained.
- File system: ESP VFS FAT; the mount point is defined by the BSP; `format_if_mount_failed=false`, up to 5 open files, and an allocation unit of 16 KiB.
- Software: ESP-IDF `sdmmc_cmd`, `esp_vfs_fat_sdmmc_mount`, and FatFs.
- Key source: `idf-code/Lesson08-SD_Card_File_Reading/peripheral/bsp_sd/bsp_sd.c`.

```c
host.max_freq_khz = 10000;
slot.clk = GPIO_NUM_43; slot.cmd = GPIO_NUM_44;
slot.d0 = GPIO_NUM_39; slot.width = 1;
```

### 6.2 External QSPI Flash

- The W25Q128JVSIQ is a 128 Mbit (16 MiB) device connected to the ESP32-P4’s dedicated Flash interface.
- The code does not provide a raw SPI driver or partition capacity configuration. The actual Flash mode/frequency/size must be jointly confirmed from the production `sdkconfig`, boot log, and `partitions.csv`.
- Multiple examples use custom `partitions.csv` files; do not assume that all course projects use identical partition layouts.

## 7. USB and UART

### 7.1 USB2 Type-C Device Port (J16)

- The P4’s native DP/DM signals connect to `USB2_D+/-` and use the internal PHY (`external_phy=false`).
- Lesson06 enumerates as a TinyUSB HID boot mouse; IN endpoint `0x81`, maximum packet size 16 bytes, polling interval 10 ms.
- The configuration descriptor declares 100 mA and remote wakeup; high-speed and full-speed operation use the same configuration descriptor.
- Software dependencies: `espressif/esp_tinyusb ^1.1` and the TinyUSB HID class.
- The Type-C CC resistors and VBUS are hardware functions and are not controlled by code.

### 7.2 Download/Logging Port (J1 + CH340K)

- The CH340K UART connects through series resistors to P4 UART0: GPIO37 TXD0 and GPIO38 RXD0.
- DTR/RTS control `SPI_BOOT` and `P4_RST_EN` through the UMH3NTN automatic download circuit.
- Pressing K3 pulls BOOT low; pressing K4 pulls CHIP_PU/RESET low.
- This path is used by the ROM bootloader, ESP-IDF console, and flashing tools; there is no application BSP.

### 7.3 Expansion UART

- Successfully tested in Lesson04: uses **UART_NUM_2**, 115200 baud, 8 data bits, no parity, 1 stop bit, no flow control, and the default source clock; GPIO47 TX, GPIO48 RX; 2048-byte RX buffer.
- The drawings name GPIO47/48 as TXD1/RXD1 and route them to J2. The difference between the UART controller number and network names does not constitute an electrical conflict: the ESP32 GPIO Matrix allows UART2 to be mapped to these two pins.
- The header file additionally defines GPIO34/33 for the input-side UART, but the current `uart_init()` does not initialize the second channel.

## 8. Audio

### 8.1 PDM Microphone

- Device: U176 LMA3526B381; PDM clock GPIO24, PDM data GPIO26, and the L/R selection network connected to GPIO20.
- I2S0 RX master; 16 kHz, 16-bit, mono, left slot, 8× down-sampling, BCLK divider 8.
- High-pass filter enabled, cutoff frequency 35.5 Hz, amplify=1; DMA uses 6 descriptors × 256 frames.
- GPIO20 is not configured in code, so channel selection depends on the hardware default level. Its pull-up or pull-down must be explicitly defined when revising the board or replacing the microphone.
- Software: ESP-IDF new I2S driver `i2s_pdm_rx`, FreeRTOS, and SPIRAM heap.

### 8.2 Digital Audio Output and Amplifier

- I2S1 TX master: GPIO21 LRCLK/WS, GPIO22 BCLK, GPIO23 SDATA; MCLK is unused.
- 16 kHz, 16-bit, stereo, both slots, MCLK multiple 256, DMA 6×256.
- GPIO30 is `AUDIO_OUT_SD`/amplifier control, configured as a push-pull output with no pull-up or pull-down and **active-low**: `set_Audio_ctrl(true)` actually writes 0.
- The drawings include two NS4168 digital amplifiers and an NS4263B analog amplifier path. The currently tested code verifies the GPIO21/22/23 digital stream and GPIO30 global control, but does not include independent register configuration for the NS4263B.

```c
i2s_std_config_t audio = {
    .clk_cfg.sample_rate_hz = 16000,
    .gpio_cfg = {.bclk = 22, .ws = 21, .dout = 23},
};
gpio_set_level(30, !enable); // active-low
```

### 8.3 ES7210 / ES8311

- The ES7210 four-channel ADC (IC5) and ES8311 codec (IC6) both appear in the schematic and connect to I2C control and I2S/TDM clock/data networks.
- The repository contains no I2C register initialization, gain, sample-rate, or power-sequencing code for either device; their status is therefore C.
- During porting, do not mistake the Lesson11 PDM microphone path as verification of the ES7210, and do not mistake direct I2S amplifier playback as configuration of the ES8311.

## 9. Camera

- The camera connects through FPC3. The drawings provide two-lane MIPI-CSI D0/D1/CLK differential pairs, SCCB, XVCLK, RESET, 1.8 V digital power, and 2.8 V analog power.
- The project provides and references a custom image parameter file for the SC2336: `idf-code/Lesson13-Camera_Real-Time/peripheral/bsp_camera/sc2336_custom.json`; actual sensor identification is still performed at runtime by `esp_video`/sensor driver.
- SCCB: I2C port 1, SDA GPIO12, SCL GPIO13, 100 kHz, with internal pull-ups enabled. The drawings show BSS138-type level shifting to `CSI_SDA/SCL`.
- Initialization passes the SCCB handle to `esp_video_init`; `reset_pin=-1`, `pwdn_pin=-1`. The drawings include a CSI_RESET circuit, but the current application does not directly drive the reset pin.
- Video device: `ESP_VIDEO_MIPI_CSI_DEVICE_NAME`, V4L2 API; the application queries the dimensions at startup and forces RGB565. The maximum buffer count is 10, while the example actually uses double buffering.
- Dependencies: ESP-IDF >=5.4.0, `esp_cam_sensor ^1.2.0`, `esp_sccb_intf ^0.0.5`, `esp_video ^1.1.0`, and LVGL.
- Resolution and frame rate are negotiated by the sensor driver; the application code does not specify fixed values. Maintenance documentation must not state a fixed 1024×600 resolution or fixed fps.

## 10. Wi-Fi Coprocessor

- Onboard IC1 is an ESP32-C6-MINI-1-N4 powered by 3.3 V; P4 GPIO32 connects to C6_EN.
- The P4 and C6 use 4-bit SDIO: GPIO14 D3, 15 D2, 16 D1, 17 D0, 18 CLK, and 19 CMD.
- Software does not directly bit-bang these GPIOs. They are managed by `espressif/esp_hosted ~2.7.0` and `esp_wifi_remote ^0.16.1`; the upper layer continues to call the `esp_wifi`/`esp_netif` APIs.- Lesson16 BSP supports STA/AP. The example AP defaults to SSID `ESP32-P4-WIFI`, password `12345678`, channel 6, and a maximum of 4 connections.

- Lesson17 provides three modes: station, softAP, and softAP+station. The connection credentials are example configurations and should not be hardcoded into production firmware.
- The C6 UART0 and BOOT/EN circuits are used for debugging and firmware download. No script for directly flashing C6 firmware was found in this project. During maintenance, preserve the matching hosted slave firmware version.

## 11. Sensors and External Modules

### 11.1 DHT20 Temperature and Humidity Sensor

- Connected externally through J13 and shares I2C0: GPIO45 SDA, GPIO46 SCL, 3.3 V, and GND; 7-bit address `0x38`, 400 kHz.
- During initialization, read status using `0x71`; reset registers `0x1B/0x1C/0x1E` if the device is not calibrated.
- Measurement command: `0xAC 0x33 0x00`; wait at least 80 ms, then poll the busy bit with a timeout of 1000 ms.
- Read 7 bytes and perform CRC-8 using polynomial `0x31`; output the 20-bit raw humidity and temperature values.
- Software: custom BSP + ESP-IDF `i2c_master`/FreeRTOS.

### 11.2 SX1262 LoRa

- SPI3 host, mode 0, 8 MHz, automatic DMA; SCK8, MISO7, MOSI6, CS/NSS10, BUSY9, DIO1/IRQ27, and NRST28.
- RadioLib 7.2.1 parameters: 915.0 MHz, 125 kHz bandwidth, SF7, CR 4/7, private sync word, 22 dBm, preamble length 8, and TCXO voltage 1.6 V.
- Reception uses a GPIO IRQ callback, `setPacketReceivedAction()`, boosted gain, and continuous receive.
- The 915 MHz frequency and 22 dBm transmit power must be adjusted according to regulations in the region of sale, the antenna, and the module version. Do not use these settings directly in regions where they are not permitted.

### 11.3 nRF24

- SPI3 mode 0, 8 MHz; SCK8/MISO7/MOSI6, IRQ9, CE27, and CS28.
- RadioLib parameters: 2400 MHz, 250 kbps, frequency offset 0, and 5-byte address width; pipe address `{0x01,0x02,0x11,0x12,0xFF}`.
- TX and RX parameters must match exactly. SX1262, nRF24, and UART passthrough cannot use GPIO27/28 simultaneously within the same task.

## 12. Power Management and Non-Programmable Hardware

### 12.1 Input, Battery, and Charging

- Power inputs include Type-C VBUS, battery connector J3, and power switch SW1; MOSFETs implement power-path switching.
- U2 TP4059 connects `VCC5V_IN` and `VBAT`, with the `CHG/STD` status signals routed to the STC8H1K08.
- The STC8 reads `ADC_VBAT` and drives the red/green charging indicators. The repository contains no STC8 source code or firmware, and the ESP32-P4 application provides no battery-level interface.
- Therefore, charge termination, charging current, LED polarity, and similar characteristics can only be confirmed by jointly examining the schematic component values and STC8 firmware; they cannot be inferred from the P4 examples.

### 12.2 Main Power Rails

| Function | Main Components | Output/Net | Software Control |
|---|---|---|---|
| Main 3.3 V / 1.2 V | TLV62569/MT3406-class DC/DC | VDD_3V3, core 1.2 V | Hardware startup; the P4 internal LDO/DC-DC is managed separately by the chip startup code |
| Camera digital supply | ME6211C18 | DOVDD_1V8 | No application control |
| Camera analog supply | ME6211C28 | AVDD_2V8 | No application control |
| Audio 3.3 V | ME6211C33 | AUDIO_VDD_3V3 | No application control |
| LCD backlight | MT9201 | LED+/LED- | GPIO31 EN/PWM, with additional power gating through GPIO29 |
| LCD bias | MT3608/discrete circuitry | 9.6 V, 18 V, -6 V, VCOM | No register-based driver |

## 13. Schematic and Code Discrepancies

| ID | Item | Schematic/Kconfig | Verified Code | Resolution and Possible Cause |
|---|---|---|---|---|
| D01 | Wireless control pins | Wireless Kconfig configurations default IRQ/CE/RST/CS to GPIO53/54 | Headers hardcode GPIO27/28 | **Use the headers/code as authoritative**. The Kconfig macros are not referenced by the headers and represent an unsynchronized configuration or definitions for an older expansion board |
| D02 | UART passthrough pins | Wireless Kconfig defaults to TX53/RX54 | Headers hardcode TX27/RX28 | **Use the code as authoritative**; these pins are also mutually exclusive with the wireless control pins |
| D03 | UART controller name | Schematic net names are TXD1/RXD1 | Lesson04 uses UART_NUM_2 mapped to 47/48 | This is valid through the GPIO Matrix; when porting, select the controller based on the GPIOs rather than the net-name suffixes |
| D04 | GPIO48 LED example | Schematic assigns GPIO48 to RXD1/J2 RX | Lesson02 treats GPIO48 as an active-high LED | **Do not use this as the baseline for the onboard LED**. It is likely an external LED, leftover course material, or a comment error; it will disrupt UART RX |
| D05 | LCD backlight | GPIO31=`LCD_BK_EN`, GPIO29=`LCD_BK_POWER` | BSP uses PWM on GPIO31 only | Use GPIO31 for current brightness control; GPIO29 must be verified for full shutdown |
| D06 | LCD reset | Schematic connects GPIO41 to LCD_RESET | EK79007 configuration uses `reset_gpio_num=-1` | Use the verified command-based reset; hardware reset capability is not used by the software |
| D07 | Camera reset | Schematic includes CSI_RESET/level shifting | `reset_pin=-1`, `pwdn_pin=-1` | Rely on the sensor/onboard default sequencing; GPIO reset may need to be added when changing modules |
| D08 | Automatic SD formatting | README states that the card is automatically formatted if mounting fails | Code uses `format_if_mount_failed=false` | **Use the code as authoritative**; the README description is outdated, and automatic formatting does not occur in practice |
| D09 | Audio component scope | Schematic includes ES7210/ES8311/multiple amplifiers | Examples only verify PDM RX and direct I2S TX/global amplifier control | Do not claim that codec register drivers have been verified |
| D10 | I2C pull-ups | Code enables internal pull-ups | Schematic includes external pull-ups and level shifting | Use the external hardware pull-ups as the reliable baseline; internal pull-ups are supplemental only and must not be used for bus calculations |

## 14. Risks and Precautions

### 14.1 High Risk

1. **GPIO27/28 multifunction conflict**: SX1262, nRF24, and wireless UART passthrough use the same GPIOs, and GPIO27/28 are also the P4_TXD2/P4_RXD2 nets. Only one function can be enabled per build. Refactoring to a single board pin configuration with compile-time mutual-exclusion checks is recommended.
2. **GPIO48 mislabeled as an LED**: This pin is the J2 UART RX line. Running Lesson02 continuously toggles the serial receive line, potentially disrupting peripherals or even causing contention.
3. **Misinterpretation of 5 V interfaces**: J10 provides `+5V_IN`, and J9 provides 5 V power, but ESP32-P4 GPIO logic generally remains in the 3.3 V domain. Do not apply 5 V logic signals without level shifting.
4. **LCD high-voltage rails**: The backlight, VGH, VGL, and AVDD include approximately 18 V, -6 V, and 9.6 V rails. When measuring or modifying the board while powered, prevent shorts to the MIPI/3.3 V nets.
5. **Regional RF regulations**: The SX1262 example is fixed at 915 MHz/22 dBm. Production settings must be adjusted according to the country, module certification, antenna gain, and duty-cycle requirements.

### 14.2 Medium Risk

1. I2C0 is shared by the GT911, DHT20, and external J13 connector. Device addresses, bus capacitance, pull-up values, and hot-plugging all affect stability at 400 kHz.
2. The RadioLib HAL calls `gpio_install_isr_service()` during each initialization. When multiple drivers are initialized sequentially, it may return already installed; a unified BSP should install the ISR service only once.
3. The RadioLib HAL exclusively initializes and releases `SPI3_HOST`. If other devices share SPI3, bus ownership must be centrally managed.
4. The backlight duty-cycle mapping does not clamp `brightness > 100`.
5. The SD interface runs only at 10 MHz in 1-bit mode, resulting in lower performance but greater fault tolerance. Before switching to 4-bit/high-speed mode, confirm the schematic pull-ups on DATA1/2/3 and the PCB signal quality.
6. Audio is fixed at 16 kHz. Before playing MP3/WAV files, resample them or reconfigure I2S to match the file’s sample rate; otherwise, the pitch and playback speed will be incorrect.
7. Camera dimensions are dynamically negotiated through V4L2. Display buffer allocation must account for the actual width, height, stride, and RGB565 size.

### 14.3 Software Maintenance Risks

1. Wireless Kconfig and the headers contain duplicate definitions with inconsistent values. A single authoritative board configuration should be retained.
2. The Lesson02/04 READMEs retain generic upstream examples and do not match the current code. Use the source code as authoritative during maintenance.
3. The repository lacks a single locked `sdkconfig.defaults`; menuconfig settings on different development machines may produce different pin, display, and Wi-Fi behavior.
4. Each Lesson contains a copy of the same BSP, so fixes must be synchronized across multiple copies. Extracting it into `components/board_bsp` is recommended.
5. The ESP32-C6 hosted slave firmware and version-matching instructions have not been preserved. Future `esp_hosted` upgrades may cause host/slave protocol incompatibilities.

## 15. Recommended Initialization Sequence

1. Start ESP-IDF, NVS, and the event loop, and wait as necessary for power stabilization;
2. Initialize shared I2C0 (GPIO45/46);
3. Initialize the LCD MIPI-DSI interface, then initialize LEDC on GPIO31, and finally increase the backlight gradually;
4. Initialize the GT911 and register the LVGL input device;
5. Initialize SDMMC, I2S audio, and camera SCCB/CSI as required by the product;
6. If Wi-Fi is enabled, start the ESP32-C6 hosted link before calling `esp_wifi`;
7. Select only one of SX1262, nRF24, and UART transponder, as they use SPI3/GPIO27/28;
8. Perform shutdown in the reverse order: mute audio, turn off the backlight, and stop streaming before releasing buses and disabling power gates.

## 16. Driver Porting Checklist

- [ ] Set the target to ESP32-P4 and record the exact ESP-IDF version.
- [ ] Create a single `board_pins.h`, copy the GPIO assignments verified in this document, and do not directly reuse outdated Kconfig defaults.
- [ ] Pin the versions of the EK79007, GT911, LVGL, esp_hosted, esp_wifi_remote, and RadioLib components.
- [ ] Confirm Flash/PSRAM capacities, modes, frequencies, and the partition table.
- [ ] Verify the default levels of GPIO29, 30, 31, and 32 after power-on to prevent backlight flashing, amplifier pops, or C6 resets.
- [ ] Perform an I2C scan to confirm the GT911 address and the addresses of all external sensors.
- [ ] Use an oscilloscope to verify DSI power sequencing, the I2S sample rate, SD at 10 MHz, SPI at 8 MHz, and SCCB at 100 kHz.
- [ ] Add compile-time resource conflict checks for UART, wireless, and expansion GPIOs.
- [ ] For each function, preserve the “firmware commit/configuration, board version, test procedure, and passing logs.”
- [ ] After RF and power changes, repeat EMC, ESD, thermal, and regulatory testing.

## 17. Evidence Index

| Evidence | Path |
|---|---|
| EAGLE schematic source file | `1.2/ESP32-P4 Display 9.0 inch V1.2.sch` |
| Schematic PDF | `1.2/ESP32-P4 Display 9.0 inch V1.2.pdf` |
| PCB source file | `1.2/ESP32-P4 Display 9.0 inch V1.2.brd` |
| UART | `idf-code/Lesson04-Serial_port_usage/peripheral/bsp_uart/` |
| GT911/I2C | `idf-code/Lesson05-Touchscreen/peripheral/` |
| USB HID | `idf-code/Lesson06-USB2.0/peripheral/bsp_usb/` |
| LCD/backlight | `idf-code/Lesson07-Turn_on_the_screen/peripheral/bsp_illuminate/` |
| MicroSD | `idf-code/Lesson08-SD_Card_File_Reading/peripheral/bsp_sd/` |
| DHT20 | `idf-code/Lesson10-Temperature_and_Humidity/peripheral/` |
| PDM/I2S audio | `idf-code/Lesson11-Playback_After_Recording/peripheral/` |
| Camera | `idf-code/Lesson13-Camera_Real-Time/peripheral/bsp_camera/` |
| SX1262 | `idf-code/Lesson14_RX_SX1262_Wireless_Module/peripheral/bsp_wireless/` |
| nRF24 | `idf-code/Lesson15_RX_nRF2401_Wireless_RF_Module/peripheral/bsp_wireless/` |
| ESP32-C6 Wi-Fi hosted | `idf-code/Lesson16_Get_weather_via_WiFi/`, `idf-code/Lesson17-Wi-Fi_function/` |

## 18. Change Log

| Version | Date | Description |
|---|---|---|
| V1.0 | 2026-07-30 | Initial release; completed cross-validation of the schematic, PCB net naming, and drivers in 17 ESP-IDF examples |