# COS-102 hat for ESP32-S31-Function-CoreBoard-1

KiCad project for a hat that plugs onto the CoreBoard's J2 header and drives the
Delta COS-102 LED sign's IDC "IN" connector. It is built from `SCHEMATIC.md`, with
the GPIO map changed for the S31 board.

Files: `cos102_s31_hat.kicad_pro` / `.kicad_sch` / `.kicad_pcb`, plus a project
symbol library (`cos102_hat.kicad_sym`, the CoreBoard J2 symbol with real pin
names) and footprint library (`cos102_hat.pretty`, the button access hole).
Saved in KiCad 7 format, so KiCad 7, 8, 9 and 10 all open it (newer versions
upgrade it on save).

## GPIO map: firmware must change

The ESP32-S3 pins in `pins.h` (4-8, 15-18) are not on the S31 board's J2,
except GPIO4, and GPIO4 is also the Ethernet PHY interrupt (`ETH_INTN`, through
R83). The hat uses nine free pins next to each other on J2:

```c
#define PIN_CLK    47   // J2.13 -> U8A  -> R1  -> P1.10
#define PIN_R      48   // J2.14 -> U8D  -> R4  -> P1.4   (+ U10B -> R10 -> P1.1 copy)
#define PIN_G      45   // J2.15 -> U8C  -> R3  -> P1.3
#define PIN_B      46   // J2.16 -> U8B  -> R2  -> P1.2
#define PIN_LAT    43   // J2.17 -> U9A  -> R5  -> P1.9
#define PIN_GCLK   44   // J2.18 -> U9B  -> R6  -> P1.11
#define PIN_ADDR_A 40   // J2.19 -> U9C  -> R7  -> P1.8
#define PIN_ADDR_B 42   // J2.20 -> U9D  -> R8  -> P1.7
#define PIN_ADDR_C 39   // J2.21 -> U10A -> R9  -> P1.6
```

CLK and the three data lines share U8, as in `SCHEMATIC.md`.

Pins deliberately avoided, from the CoreBoard schematic and the ESP32-S31 IO guide:

| GPIO | Why |
|---|---|
| 4 | Also wired to the Ethernet PHY interrupt (ETH_INTN) through 0 Ohm R83 |
| 36, 37 | Strapping (VDD_SPI voltage, JTAG select) |
| 60 | Strapping + on-board RGB LED |
| 61 | Strapping (BOOT) + BOOT button |
| 58, 59 | UART0 to the CP2102N (console, `oc fast` etc.) |

Spare and free on J2: GPIO0-3, 20-25 (SD pins), 35, 38, 49.

## Power

Same as `SCHEMATIC.md`: J1 (display PSU 5 V) -> F1 2 A slow -> D1 TVS / C5 ->
FB1+FB2 -> L1 47 uH + C6 1000 uF -> FB3 -> buffers, and -> D2 SS34 -> C8 470 uF
-> CoreBoard 5V pins (J2.39/40).

On the CoreBoard, the 5V header pins are `VCC_5V`, the output of the LM66200
ideal-diode OR of both USB-C ports. That rail also feeds the 3.3 V buck, the
USB-A host switch (up to 500 mA) and the speaker amp. D2 stops the board's USB
from powering the panel. USB can stay plugged in for flashing.

GND_IN (J1 pin 2, D1, C5, FB2 pin 1) has its own copper island on the top layer
(`GND_IN_island` zone). The main GND pours on both layers start after FB2.

## Mechanics

- 65 x 55 mm, same outline as the CoreBoard. 3.5 mm corner radius, 4x M2.5 holes on the board's 58 x 48 mm grid.
- J2 is a 2x20 female socket (8.5 mm body) on the BOTTOM side. Every pad was
  checked against the CoreBoard header positions from Espressif's DXF; pin 1 is
  on the inner row at the left end.
- Stack height: 2.54 mm header plastic + 8.5 mm socket = 11 mm. Use M2.5 x 11 mm standoffs.
- Cut-out over the RJ45 (x >= 43.5 mm, y 17-44.5 mm). The CoreBoard's USB-A is
  a side-mount part standing on edge, so both are taller than 11 mm.
- 3.2 mm holes over the BOOT and RST buttons, so you can press them with a toothpick.
- Copper keep-out (both layers) along the left edge, where the module's PCB antenna hangs off the CoreBoard.
- `User.Drawings` shows the CoreBoard RJ45, USB-A and module outline for reference.

## Check before ordering

1. **P1 pin 1** against the ribbon's triangle mark (unchanged from `SCHEMATIC.md`).
2. **IDC pins 5, 13, 15** are left unconnected until they have been metered.
3. Panel 5 V current. Bench test with a current-limited supply first.

Already checked: ERC is clean (0 errors, 0 warnings) in your KiCad. The board
has no courtyard overlaps. J1's wire openings face the bottom board edge
(checked against its 3D model). J1 stands about 13.8 mm tall above the hat.

## Layout and routing

P1 (the ribbon to the panel) sits on the bottom edge next to J1, so both cables
plug in along the same edge. Pin 1 is at the left, in the same orientation as
before. The fuse F1 sits in the gap between P1 and J1. The input TVS/bypass/beads
(D1, C5, FB1, FB2) sit just above them. L1 and C6 are in the middle of the board.

The board is routed on 2 layers. Most tracks are on top. The bottom is a GND
plane, cut only by about 176 mm of short jumpers.

- 68 vias. Each buffer has a GND bus under its body that joins pins 1/4/7/10/13,
  with two vias to the plane. Each decoupling cap is tied straight to pins 13/14.
  Every other SMD GND pad has its own via.
- GND_IN (J1 pin 2, D1, C5, FB2 pin 1) is its own short 0.8 mm track and never touches the plane.
- Track widths: Power 0.8 mm, +5V_BUF 0.5 mm, panel side 0.3 mm, 3.3 V side 0.25 mm.
- Clock/data path lengths from J2 to P1: CLK 60 mm, G 53 mm, B 65 mm, R 74 mm
  (R includes the pin-1 copy branch). That's under 0.1 ns of skew, negligible at 18 MHz.
- DRC (zones filled): 0 unconnected, 0 clearance errors at 0.2 mm for signals and
  0.25 mm for power/GND. Left over: library silkscreen lines (P1's pin-1 marker,
  J1's front edge) that run past the board edge and get trimmed.

It was autorouted with a small grid router, because Freerouting isn't installed
here. Press B after any edit to refill the zones.

## JLCPCB (standard 2-layer, 1 oz) check

| Rule | JLCPCB minimum | This board |
|---|---|---|
| Track width / spacing | 0.10 / 0.10 mm | 0.25 / 0.20 mm |
| Via hole / diameter | 0.15 / 0.25 mm | 0.30 / 0.60 mm (power 0.40 / 0.80) |
| PTH annular ring | 0.18 mm (0.25 recommended) | 0.35 mm |
| Hole to hole, vias | 0.20 mm | 0.61 mm |
| Hole to hole, via to pad | 0.45 mm | 0.75 mm |
| Pad to pad | 0.15 mm | 0.20 mm |
| PTH hole to track | 0.28 mm | 0.55 mm |
| NPTH to copper | 0.20 mm | 0.25 mm |
| Copper to board edge | 0.20 mm | 0.50 mm |
| Solder mask bridge | 0.10 mm (green) | 0.67 mm (SOIC-14 pins) |
| Min NPTH | 0.50 mm | 3.2 mm |
| Silkscreen text height | 1.0 mm | 1.0 mm |
| Silkscreen line width | 0.15 mm | 0.12 mm in KiCad library footprints* |

\* KiCad's standard library footprints draw silkscreen outlines at 0.12 mm.
JLCPCB prints them, though possibly thinner or with small gaps. This is cosmetic
only and doesn't affect fabrication.

For JLCPCB assembly you would also need LCSC part numbers in the BOM. The
footprints are all standard. J1, P1, J2, C6 and C8 are through-hole parts, so
solder them by hand (or pay for THT assembly).

## BOM (JLCPCB / LCSC, checked against the JLCPCB catalogue of 2026-11-05)

`jlcpcb_bom.csv` is ready to upload to JLCPCB assembly. Every symbol and footprint also
carries an `LCSC` field, so the KiCad JLCPCB plugins pick the numbers up as well.

| Ref | Part | LCSC | Type | Stock | Price (1+) |
|---|---|---|---|---|---|
| U8, U9, U10 | TI SN74AHCT125DR, SOIC-14 | C155176 | Extended | 2,513 | $0.40 |
| R1-R10 | UNI-ROYAL 0603WAF330JT5E, 33 R 1 % 0603 | C23140 | Basic | 5.07 M | $0.0025 |
| C1-C3, C5, C7, C9 | YAGEO CC0603KRX7R9BB104, 100 nF 50 V X7R | C14663 | Basic | 50 M | $0.014 |
| C4 | Samsung CL21A106KAYNNNE, 10 uF 25 V X5R 0805 | C15850 | Basic | 3.7 M | $0.069 |
| L1 | Bourns SRR1260-470M, 47 uH, Isat 2.6 A, 86 mOhm | C1329363 | Extended | 600 | $1.48 |
| FB1, FB2 | Murata BLM31PG121SN1L, 1206, 3.5 A, 20 mOhm | C85841 | Extended | 49,437 | $0.16 |
| FB3 | Murata BLM18PG221SN1D, 0603, 1.4 A | C80165 | Extended | 298,012 | $0.015 |
| F1 | Littelfuse 0452002.MRL, 2 A slow-blow | C142852 | Extended | 12,284 | $0.33 |
| D1 | MDD SMBJ5.0A, unidirectional TVS | C113974 | Extended | 178,091 | $0.059 |
| D2 | MDD SS34, SMA | C8678 | Basic | 4.0 M | $0.035 |
| C6 (THT) | Rubycon 16YXF1000MEFC10X20, 1000 uF 16 V, 10x20, 78 mOhm | C110138 | Extended | 6,499 | $0.20 |
| C8 (THT) | Nichicon UHE1A471MPD1TD, 470 uF 10 V, 8 mm | C1579341 | Extended | 1,489 | $0.14 |
| J1 (THT) | Phoenix Contact 1729128 (MKDS 1,5/2-5,08) | C7509570 | Extended | 1,306 | $0.54 |
| P1 (THT) | XFCN BH254V-16P, 2x8 box header | C492443 | Extended | 114,323 | $0.16 |
| J2 (THT, bottom) | CONNFLY DS1023-2*20SF11, 2x20 female, 8.5 mm | C132132 | Extended | 2,884 | $0.49 |
| H1-H4 | M2.5 mounting holes, no part | | | | |

- SMT parts cost about $3.60 per board. Six SMT parts are Extended (U, L1, FB1/2, FB3, F1, D1),
  which adds about $18 in setup fees per order ($3 per unique Extended part).
  None of them has a Basic equivalent in the catalogue.
- The through-hole parts (C6, C8, J1, P1, J2) cost about $1.50. Solder them by hand, or
  pay for JLCPCB's THT assembly. J2 goes on the bottom side.
- Lowest stock is L1 (600). Stock moves, so re-check before ordering.
- Cheaper J1 alternative: KF128-5.08-2P-AA (C474952, $0.19, 62k in stock), same 5.08 mm
  pitch. Check its pin and body against the MKDS footprint before swapping.
- The parts file (CPL) and Gerbers are best produced with the JLCPCB fabrication
  plugin in KiCad, which applies JLCPCB's rotation offsets per footprint.
