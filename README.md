# ESP32-Custom-Development-Board-with-ESP32-WROOM-32D-Module
Designed a custom 4 layer ESP32 development board around the ESP32 WROOM 32D module in EasyEDA: CP2102N USB to UART with transistor auto reset, dual buck power (12V/USB Type C to 5V to 3.3V) with ideal diode ORing, and a board edge antenna keep out. The board works reliably in DIO mode at 40 MHz and detects WiFi networks from −9 dBm to −95 dBm.
# ESP32 WROOM 32D Custom Development Board

> Hardware design and learning notes for a custom ESP32 development board built around the ESP32 WROOM 32D module.

**Author:** Devansh Sharma, 1st Year Integrated MSc, NISER Bhubaneswar
**Version:** V2.0

## Overview

This board uses the precertified ESP32 WROOM 32D module, which already contains the flash, 40MHz crystal, RF matching network and PCB antenna. That keeps the focus on the circuits around the module: power architecture, USB to UART, auto reset, grounding and antenna placement, all on a 4 layer PCB.

**Supports:**

1. USB Type C and external VIN (for example 12V) power input through a DC barrel jack or a 2 pin header
2. Automatic reset and boot control circuit for one click flashing
3. Onboard module antenna (MIFA) for 2.4GHz WiFi and Bluetooth
4. 4MB integrated SPI flash

## ESP32 WROOM 32D Module

Even with the flash, clock and RF inside the module, stable operation still depends on power integrity, reset behaviour, boot configuration and grounding working together.

### Power Supply and Decoupling

The module needs 3.0V to 3.6V from a supply that can deliver at least 500mA. The 3V3 pin is decoupled with **100nF** for high frequency noise and **22µF** for WiFi burst current. Both capacitors sit right at the pin with short vias to the ground plane. Without this, WiFi current spikes cause voltage dips that trigger the brownout detector.

### Enable (EN) Pin

EN must be HIGH for normal operation. It is pulled up with a 10kΩ resistor and filtered with a 100nF capacitor to GND for a stable power up delay. It connects to both the auto reset circuit and a RESET push button, which has its own 100nF capacitor for debouncing.

### Boot Strapping

<table>
<tr><th>GPIO0 State</th><th>Boot Mode</th></tr>
<tr><td>LOW</td><td>Flashing / programming mode</td></tr>
<tr><td>HIGH</td><td>Normal execution</td></tr>
</table>

GPIO0 has no external pull up and relies on the internal pull up of the ESP32. It is driven LOW through DTR during upload, or by the BOOT push button, which has a 100nF capacitor for debouncing.

### Grounding and Thermal Pad

The three module GND pins (1, 15 and 38) and the exposed ground pad are all tied to the ground plane through multiple vias. The pad acts as the heat path, the central ground reference and the noise return path. A solid GND plane runs under the whole module.

## Flash Interface

The 4MB SPI flash is inside the module and connects to the ESP32 through the SD_DATA, SD_CLK and SD_CMD pins (GPIO6 to GPIO11), so no external flash is needed.

These six pins (SD0 to SD3, CMD and CLK) are still broken out on the headers, but **only for probing**. Because the ESP32 executes code directly from flash (XIP), using them as GPIO would stop instruction fetch and crash the chip.

## USB to UART Interface

**IC:** CP2102N (Silicon Labs, 28 pin QFN)

It bridges USB and UART for serial communication, firmware upload and debug logging, and provides the DTR and RTS signals for auto reset.

**Signal chain:** USB Type C → ESD protection (USBLC6) → 22Ω series resistors → CP2102N → ESP32

The ESD diode must come before the IC, and the series resistors match the line to about 90Ω differential. CP2102N TX connects to ESP32 RX and vice versa.

**Key pins:**

1. Powered from the 3.3V rail with 100nF + 4.7µF decoupling on VDD and VREGIN
2. VBUS sensed through a 22.1kΩ / 47.5kΩ divider for USB detection
3. RSTb pulled HIGH through 2kΩ
4. SUSPENDb tied to a 10kΩ resistor as per the reference design
5. TX and RX status LEDs

## Auto Reset and Boot Control

Two **SS8050** NPN transistors translate the DTR and RTS signals from the CP2102N into controlled pull downs on GPIO0 and EN.

<table>
<tr><th>Signal</th><th>Controls</th></tr>
<tr><td>DTR</td><td>GPIO0 (boot mode)</td></tr>
<tr><td>RTS</td><td>EN (reset)</td></tr>
</table>

**Upload sequence:**

1. DTR pulls GPIO0 LOW
2. RTS toggles EN to reset the chip
3. ESP32 enters the bootloader
4. Both lines return HIGH and the new firmware runs

10kΩ base resistors limit transistor base current. DTR and RTS are not brought out to a header, so they can only be probed at these base resistors.

## Power Block

A dual input, multi stage architecture produces a clean 3.3V rail from either USB Type C or external VIN.

### Power Flow

```
USB Type C (5V) → Polyfuse → Ferrite Bead ──────────────→ USB_5v ────┐
                                                                     ├→ LM66100 ×2 (Ideal Diode OR) → 5v_rail → TPS62162 Buck → 3.3v_rail
External VIN (12V) → TPS62163 Buck → Polyfuse ──────────→ 5v_input ──┘
```

### USB Type C Path

Polyfuse for overcurrent protection, a 600Ω at 100MHz ferrite bead to block cable noise, 100nF + 4.7µF decoupling, and **5.1kΩ resistors on CC1 and CC2**, which are mandatory for the host to supply power.

### External VIN Path

Input comes from a DC barrel jack or the VIN pin of a 2 pin header. It is decoupled with 47µF + 10µF + 100nF and fed to a TPS62163 buck, whose EN is pulled up to the input through 100kΩ. The output stage uses a 2.2µH inductor, 22µF + 100nF + 47µF capacitors, a 100kΩ pull up on PG and a polyfuse to produce 5v_input.

### Power Selection

Two **LM66100** ideal diodes perform power ORing. They automatically select the higher 5V source with near zero voltage drop and prevent back feeding between sources.

### 3.3V Rail

A **TPS62162** buck converts 5V to 3.3V using a 2.2µH inductor, 100kΩ pull ups on EN and PG, and 47µF + 22µF + 100nF output filtering. A buck was chosen over an LDO for higher efficiency and better handling of WiFi transient loads.

### Measurement Points

No dedicated test points are provided on this board. 3.3V and GND can be measured from the headers, VIN from the 2 pin header, and 5v_rail only at the capacitor pads.

## PCB Layer Stackup

<table>
<tr><th>Layer</th><th>Function</th></tr>
<tr><td>L1 (Top)</td><td>Components and signal routing, with GND copper pour in free areas</td></tr>
<tr><td>L2</td><td>Continuous solid GND plane</td></tr>
<tr><td>L3</td><td>3.3V power plane with a few routed traces</td></tr>
<tr><td>L4 (Bottom)</td><td>Components and signal routing, with GND copper pour in free areas</td></tr>
</table>

**Why this arrangement:**

1. The solid GND plane on L2 sits directly under the top layer, giving every signal an unbroken reference and a low impedance return path.
2. The dedicated 3.3V plane on L3 distributes power with minimal resistive loss. Components connect to it through a via at the pad edge, or a short trace and then a via.
3. The GND pours on L1 and L4, stitched to L2 with vias, give an even shorter return path for surface components.
4. No plane or pour extends under the antenna, since the antenna hangs past the board edge.

## Module Placement and Antenna Keep Out

The module's antenna and RF matching network are already tuned by Espressif. Following Espressif guidelines, the module sits at the board edge with its antenna portion extending past the edge, so there is naturally no copper and no components beneath or near it. Noisy circuits like the buck converters are kept away from the module.

With a module, antenna design becomes a placement problem. As long as the keep out is respected, the pretuned antenna works without any RF tuning on the board.

## Design Decisions

<table>
<tr><th>Decision</th><th>Reason</th></tr>
<tr><td><b>WROOM 32D module over bare chip</b></td><td>Precertified flash, crystal, RF matching and antenna remove the hardest parts of the design. The trade off is fixed flash size and antenna.</td></tr>
<tr><td><b>Buck over LDO for 3.3V</b></td><td>Higher efficiency and better handling of WiFi current spikes, reducing brownout risk</td></tr>
<tr><td><b>CP2102N for USB to UART</b></td><td>Follows the official ESP32 reference design, with native DTR and RTS for auto reset</td></tr>
<tr><td><b>LM66100 over Schottky</b></td><td>Near zero voltage drop and active back feed prevention</td></tr>
</table>

## Components

<table>
<tr><th>Ref</th><th>Part</th><th>Function</th></tr>
<tr><td>U7</td><td>ESP32 WROOM 32D</td><td>WiFi + Bluetooth module with 4MB flash</td></tr>
<tr><td>U8</td><td>CP2102N (QFN28)</td><td>USB to UART bridge</td></tr>
<tr><td>U2</td><td>TPS62163DSGR</td><td>Buck regulator, VIN to 5V</td></tr>
<tr><td>U1</td><td>TPS62162DSGT</td><td>Buck regulator, 5V to 3.3V</td></tr>
<tr><td>U3, U4</td><td>LM66100DCKR</td><td>Ideal diode controllers</td></tr>
<tr><td>D1</td><td>USBLC6</td><td>USB ESD protection</td></tr>
<tr><td>Q1, Q2</td><td>SS8050</td><td>Auto reset transistors</td></tr>
<tr><td>F1, F2</td><td>Polyfuse</td><td>Overcurrent protection on USB and VIN paths</td></tr>
<tr><td>L1</td><td>Ferrite Bead</td><td>USB VBUS noise filtering</td></tr>
<tr><td>R1, R2</td><td>5.1kΩ</td><td>USB Type C CC resistors</td></tr>
<tr><td>LED1, LED2, LED3</td><td>LEDs</td><td>RX, TX and power indication</td></tr>
</table>

## Key Concepts

**Brownout Reset:** the ESP32 resets itself if the supply dips below about 2.5 to 2.7V, even momentarily. WiFi bursts can cause such dips if decoupling is weak.

**XIP (Execute In Place):** instructions run directly from flash without being copied to RAM. This is why the flash pins on the headers must never be used as GPIO.

**USB to UART Conversion:** USB uses differential, packet based signalling on D+ and D−, while UART uses simple TX/RX voltage levels. The CP2102N contains a USB engine and a UART engine that translate between the two, so the PC sees a virtual COM port and the ESP32 sees plain UART.

## Results

<table>
<tr><th>Test</th><th>Outcome</th></tr>
<tr><td>Overall</td><td>Functional first hardware implementation</td></tr>
<tr><td>Flash mode</td><td>Works normally at DIO 40MHz</td></tr>
<tr><td>WiFi reception</td><td>Detects networks from −9 dBm to −95 dBm</td></tr>
<tr><td>Power up</td><td>Slow start up issue: after first connecting the supply, a manual boot is needed once</td></tr>
</table>

**Known issues:**

1. The slow power start up issue is still unresolved.
2. The lack of dedicated test points makes bring up and auto reset debugging harder.

## Version History

<table>
<tr><th>Version</th><th>Changes</th></tr>
<tr><td>V1.0</td><td>Initial layout</td></tr>
<tr><td>V2.0</td><td>Final PCB layout, reworked because the V1.0 layout was not clean enough</td></tr>
</table>

## References

1. [ESP32 DevKitC V4 Official Schematic](https://dl.espressif.com/dl/schematics/esp32_devkitc_v4-sch.pdf)
2. [ESP32 WROOM 32 Datasheet](https://documentation.espressif.com/esp32-wroom-32_datasheet_en.pdf)
3. [TPS62160 Family Datasheet](https://www.ti.com/lit/ds/symlink/tps62160.pdf)
4. [LM66100 Datasheet](https://www.ti.com/lit/ds/symlink/lm66100.pdf)
5. [CP2102N Datasheet](https://www.silabs.com/documents/public/data-sheets/cp2102n-datasheet.pdf)
6. [USBLC6 Datasheet](https://www.st.com/resource/en/datasheet/usblc6-2.pdf)
7. Other component datasheets sourced from LCSC through EasyEDA
