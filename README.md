# Stardance — ESP32-S3 Multi-Sensor Telemetry Board

A compact, single-board data logger built around an **ESP32-S3** class module with
**GNSS positioning**, **three-axis motion sensing**, **barometric altitude**, a
**battery-backed real-time clock**, and a **high-side current/power monitor**.

The board is designed to be built cheaply and in small volumes: two copper layers,
1.6 mm FR4, no impedance-controlled traces, no castellations, no fine-pitch
microvias. Everything on it can be assembled by a standard turnkey PCBA service
such as PCBWay.


---

## Table of contents

1. [What the board does](#what-the-board-does)
2. [At a glance](#at-a-glance)
3. [Functional blocks](#functional-blocks)
4. [Board mechanicals](#board-mechanicals)
5. [Power architecture](#power-architecture)
6. [Net and interface summary](#net-and-interface-summary)
7. [Bill of materials](#bill-of-materials)
8. [Fabrication notes](#fabrication-notes)
9. [Assembly notes](#assembly-notes)
10. [Cost estimate](#cost-estimate)
11. [Bring-up and test procedure](#bring-up-and-test-procedure)
12. [Firmware interface](#firmware-interface)
13. [Known issues](#known-issues)
14. [Design rules and constraints](#design-rules-and-constraints)
15. [Revision history](#revision-history)
16. [File inventory](#file-inventory)
17. [Licence](#licence)

---

## What the board does

Stardance is a **sensor fusion and position logger**. In one small board it
combines:

- an **ESP32-S3** module with onboard antenna — the compute, Wi-Fi and BLE radio,
- a **u-blox MAX-M10S** GNSS receiver — position, velocity and precise time,
- an **ADXL372** — high-g, low-power accelerometer for shock and impact events,
- an **LSM6DSOX** — 6-axis IMU for orientation and motion context,
- a **BMP388** — barometric pressure and temperature, for altitude,
- a **PCF8563T** RTC with a 32.768 kHz crystal — timekeeping across power cycles,
- an **INA228** — 20-bit current, voltage and power monitoring,
- a **USB-C** receptacle for power and data,
- an **AP2112K-3.3** LDO for the 3.3 V rail.

The design intent is a board that can answer, simultaneously: *where am I, how
fast am I moving, how did I get knocked around, how high am I, what time is it,
and how much energy have I actually used?* That combination is useful for
wearables, sports instrumentation, drone and rocket flight recorders, field
survey equipment, and any battery-powered device where you want to log motion
plus position plus power draw over a long deployment.

Because all of the sensors sit on shared I²C and SPI buses behind a single
microcontroller, the board can be configured at runtime for either a
high-rate short-duration capture (IMU at 1.6 kHz, streaming over USB/Wi-Fi) or a
low-rate long-duration log (one GNSS fix every few minutes, everything else in
its lowest-power mode, data to flash or SD).

---

## At a glance

| Parameter | Value |
| --- | --- |
| Board outline | **54.5 mm × 86.225 mm** (47.0 cm²) |
| Copper layers | **2** (F.Cu, B.Cu) |
| Board thickness | 1.6 mm |
| Surface finish | ENIG recommended (LGA land patterns) |
| Solder mask | Both sides, 0.05 mm mask expansion |
| Silkscreen | Both sides |
| Minimum via drill | 0.20 mm |
| Minimum track width used | 0.1124 mm |
| Largest / most critical part | ADXL372, 16-pad 3.0 × 3.25 mm LGA |
| Mounting | 4 × mounting holes |
| Programming | USB-C + 2 × 5 pin header |
| Design tool | KiCad |
| Source schematic | `StardancePCB.kicad_sch` |
| PCB file | `StardancB.kicad_pcb` |

---

## Functional blocks

### Compute and radio — ESP32-S3 module with onboard antenna

An ESP32-S3-based module carrying the SoC, flash, the RF front end and a PCB
antenna. Using the module rather than a bare chip keeps the RF layout off this
board entirely, which is what makes a two-layer design viable.

The module provides the SPI and I²C buses that all sensors hang from, the USB
peripheral used for programming and data streaming, and general-purpose I/O for
interrupts and control lines.

**Layout note:** the antenna end of the module overhangs or sits at the board
edge with a copper keep-out region beneath and beside it. Do not place metal,
battery or enclosure hardware in that zone, and do not pour ground under the
antenna. When the board is installed, the antenna side should face away from the
battery.

### GNSS — u-blox MAX-M10S

A concurrent multi-constellation GNSS receiver (GPS, GLONASS, Galileo, BeiDou)
with a very low tracking current. It talks over UART or I²C depending on
configuration, and needs a clean, well-decoupled supply — hence the dedicated
`/GNSS_VCC` rail rather than sharing the noisy 3.3 V sensor rail directly.

**Design notes:**

- Keep the GNSS supply trace short and well bypassed; GNSS receivers are
  sensitive to supply noise and to nearby switching regulators.
- The receiver needs a clear view of the sky. In an enclosure, the antenna must
  not be under metal.
- If no active antenna is fitted, provide an antenna connection (u.FL/SMA) per
  the module's reference design.
- The first fix after a cold start takes tens of seconds outdoors; this is
  normal and not a fault.

### Motion — ADXL372 and LSM6DSOX

Two complementary accelerometer devices:

- **ADXL372 (U11)** — a 3-axis, high-g accelerometer intended for impact and
  shock detection. It has an internal FIFO and an event engine, and it can wake
  the host on threshold. Its interrupt line `/ADXL_INT` and chip select
  `/ADXL_CS` are routed to the module; it shares SPI with the bus on `/SPI_SCK`.
- **LSM6DSOX** — a 6-axis accelerometer plus gyroscope with an embedded
  machine-learning core, used for orientation, gait or gesture classification
  and general motion context.

The split is deliberate: the ADXL372 is the shock witness and the LSM6DSOX is
the orientation reference, so an impact event does not saturate or corrupt the
data you use for attitude.

**Package note:** the ADXL372 in this design is the
`BCCZ` 16-terminal LGA, 3.0 × 3.25 mm (JEDEC CC-16-4). It is *not*
pin-compatible with the ADXL363, which shares the package outline, so do not
substitute one for the other in assembly. Pin 5 is `MISO` on one part and
reserved on the other, pin 7 and pin 8 also differ. The land pattern used here
is `PQFN50P322X300X114-16N`.

**Mask note:** the LGA apertures on this board are drawn 1:1 with the copper
pads, i.e. **zero solder-mask expansion**. This is intentional for LGA parts —
it prevents solder bridging between adjacent pads under the body — but it means
the fab must honour a zero mask expansion on those apertures rather than
applying its own default. Two DRC warnings about mask aperture bridging on these
pads are expected and accepted.

### Pressure and altitude — BMP388

A piezo-resistive absolute pressure sensor with temperature output. Used for
barometric altitude and for vertical-velocity estimation. It is sensitive to
light and to airflow, so the datasheet calls for a small vent or a light-blocking
underfill around the port. Relative altitude from barometric pressure drifts with
weather, so treat it as short-term relative altitude, not absolute height.

### Timekeeping — PCF8563T + 32.768 kHz crystal

An I²C real-time clock with a battery-backed supply so that time survives main
power removal. The crystal is an `FC-135`-style 32.768 kHz tuning-fork part in a
3.2 × 1.5 mm SMD package.

**Layout note:** the crystal is placed close to the RTC with short traces and a
local ground guard. Do not route high-speed signals under or beside it; a
tuning-fork crystal is a high-impedance resonator and will pick up coupling.

Time discipline: the ESP32-S3's own RTC is adequate for seconds-level accuracy
but drifts with temperature. The PCF8563 plus GNSS time on fix gives you a
board that always knows the correct time within a second or so of the last fix.

### Power monitoring — INA228

A 20-bit I²C current, bus-voltage and power monitor with an integrated charge and
energy accumulator. On this board it measures through a **10 mΩ shunt** on the
battery path, so it sees the real load current of the whole board rather than a
regulated rail current. The energy accumulator makes it possible to report
"joules consumed since boot" directly, which is a much more honest power figure
than multiplying an instantaneous reading by elapsed time.

Because the shunt is only 10 mΩ, the full-scale current range is set by the
INA228's programmable gain and the shunt value — check the configured range
against your expected peak current before trusting readings, since a large
inrush (the ESP32 waking and transmitting) can exceed a conservative setting.

### Power supply — AP2112K-3.3

A 600 mA low-dropout regulator providing the 3.3 V rail. It is a linear
regulator: simple, quiet, and low part count, at the cost of dissipating
`(Vin − 3.3) × Iload` as heat. With a Li-ion cell at 4.2 V and 200 mA of load
that is about 0.18 W, which is fine in free air but worth checking in a sealed
enclosure.

The regulator is deliberately used instead of a buck converter for the sensor
rail because GNSS and precision analogue measurements care more about supply
noise than about a few tens of milliwatts.

### USB-C and connectors

- **USB-C receptacle** — `HRO TYPE-C-31-M-12`, 14-pin USB 2.0-only variant. It
  provides 5 V in, USB 2.0 data to the module, and the usual CC pull-down
  resistors (5.1 kΩ) so that a USB-C host supplies 5 V. It is *not* a USB-PD
  port.
- **2 × 5 pin header** (`Conn_02x05_Odd_Even`) — debug / programming / expansion.
- **1 × 4 pin header** (`Conn_01x04_Pin`) — secondary connector, typically used
  for a serial console or an external sensor.
- **Test points** — several, for probing the main rails and buses during bring-up.
- **4 × mounting holes.**

---

## Board mechanicals

The board outline is a single rectangle on `Edge.Cuts`:

```
gr_rect (start 97.5 95.5) (end 152 181.725)
```

giving **54.5 mm × 86.225 mm**, or 46.99 cm².

- **Enclosure allowance:** if you are quoting or modelling an enclosure, allow
  about 55 × 87 mm internal, plus 0.5 mm clearance per side for the mask and
  routing tolerance, plus room for the USB-C overhang if the connector is
  flush-mounted.
- **Standoffs:** four mounting holes; use M2 or M2.5 hardware and keep the
  corresponding keep-out area free of components on the underside.
- **Antenna zone:** keep metal and battery out of the module antenna keep-out.
- **Connector access:** USB-C is at approximately (125, 175.5) in board
  coordinates — check that your enclosure gives it a suitable cutout and that
  the cable can plug in without strain.

---

## Power architecture

```
USB-C 5 V ──► /USB_5V ──┬──► AP2112K-3.3 ──► +3V3 ──► ESP32-S3, sensors, RTC
                        │
                        └──► charge path ──► /BAT_PROT ──► /BAT_LOAD ──► 10 mΩ shunt ──► INA228
```

- **`/USB_5V`** — raw 5 V from USB, present only when a cable is connected.
- **`/BAT_PROT`** — battery node downstream of protection (over-discharge and
  over-current), so the cell cannot be damaged by a firmware bug.
- **`/BAT_LOAD`** — the switched/monitored load side.
- **`+3V3`** — the regulated rail feeding the module and all sensors.
- **`/GNSS_VCC`** — a separately decoupled branch for the GNSS receiver so that
  the module's transmit bursts do not disturb the receiver's supply.
- A **Schottky diode** provides reverse-polarity and back-feed protection between
  the supply sources, and a **600 Ω @ 100 MHz ferrite bead** filters high-frequency
  noise out of a supply branch.

Practically: the board runs from USB alone, from a battery alone, or from both
with the battery charging. When you measure power with the INA228, remember that
USB presence changes the current path — log the USB state alongside the current
reading or the numbers will not make sense.

---

## Net and interface summary

| Net | Purpose |
| --- | --- |
| `+3V3` | Regulated 3.3 V rail |
| `/USB_5V` | 5 V from USB-C |
| `/BAT_PROT` | Protected battery node |
| `/BAT_LOAD` | Monitored battery load node |
| `/GNSS_VCC` | Filtered supply for the GNSS receiver |
| `I2C_SDA` / `I2C_SCL` | Shared I²C bus (BMP388, LSM6DSOX, PCF8563T, INA228) |
| `SPI_SCK` | Shared SPI clock (ADXL372) |
| `/ADXL_CS` | ADXL372 chip select |
| `/ADXL_INT` | ADXL372 interrupt / event line |
| `/PWM2` | General-purpose PWM output |

The I²C bus carries several devices with different addresses; confirm the address
straps (the `SA0`/`SDO` style pins) do not collide before writing firmware.

---

## Bill of materials

The table below lists the significant parts. Quantities for passives are
approximate because the exact count comes from the KiCad position file; the
authoritative BOM is the one exported from the schematic.

| Ref | Part | Function | Qty | Est. unit price (10-off) |
| --- | --- | --- | --- | --- |
| U? | ESP32-S3 module, onboard antenna, SMD | MCU + Wi-Fi/BLE | 1 | $4 – $6 |
| U? | MAX-M10S | GNSS receiver | 1 | $14 – $20 |
| U11 | ADXL372BCCZ-RL7 | High-g accelerometer | 1 | $14 – $20 |
| U? | LSM6DSOX | 6-axis IMU | 1 | $4 – $7 |
| U? | BMP388 | Pressure / altitude | 1 | $2.50 – $4 |
| U? | PCF8563T | RTC | 1 | $1 – $2 |
| Y1 | FC-135 32.768 kHz, 3.2 × 1.5 mm | RTC crystal | 1 | $0.50 – $1 |
| U? | INA228 | Current / power monitor | 1 | $5 – $8 |
| U? | AP2112K-3.3 | 3.3 V LDO | 1 | $0.25 – $0.50 |
| J? | USB-C receptacle, HRO TYPE-C-31-M-12 | Power + data | 1 | $0.80 – $1.50 |
| J? | `Conn_02x05_Odd_Even` | Debug / expansion | 1 | $0.30 – $0.80 |
| J? | `Conn_01x04_Pin` | Auxiliary | 1 | $0.20 – $0.50 |
| R? | 10 mΩ shunt | Current sense | 1 | $0.30 – $1.00 |
| D? | Schottky diode | Reverse / back-feed protection | 1 | $0.20 – $0.60 |
| FB? | 600 Ω @ 100 MHz ferrite | Supply filtering | 1 | $0.05 – $0.20 |
| — | Passives: 10 k, 100 k, 100 R, 52.3 k, 5.1 k, 4.7 k, 1 µF, 100 nF, 10 nF, 15 pF, 4.7 nF, 10 µF, 22 µF | Decoupling, pull-ups, filters, timing | ~55 – 70 | $1.50 – $3.00 total |
| — | Test points, mounting holes | Bring-up / mechanical | ~9 | negligible |

**Estimated parts cost: $50 – $75 per board** at 10-off distributor pricing.
Most of that cost sits in three parts: the GNSS receiver, the ADXL372, and the
ESP32-S3 module. Everything else is small change.

Notes:

- `52.3 k` and `15 pF` are typical GNSS/oscillator or divider values — check them
  against the reference design of the specific module variant you buy.
- `5.1 k` on the USB-C CC pins is mandatory; without it a USB-C host will not
  supply 5 V.
- The 10 mΩ shunt should be a low-TCR, low-inductance part; a generic thick-film
  resistor will give you temperature- and frequency-dependent current readings.

---

## Fabrication notes

Send these with the order, or attach them as a README in the fab package:

1. **Layers:** 2, 1.6 mm thick, 1 oz copper both sides.
2. **Finish:** ENIG. The two LGA land patterns and the fine-pitch parts benefit
   from a flat finish; HASL is acceptable but makes LGA placement less reliable.
3. **Mask:** standard, **but honour the 1:1 (zero-expansion) apertures on the
   ADXL372 and LGA pads**. Do not apply a global mask expansion on those.
4. **Min drill:** 0.20 mm — this meets the standard 0.20 mm minimum CNC drill at
   the usual low-cost services. Do not request 0.15 mm drills.
5. **Impedance control:** not required.
6. **Castellations:** none.
7. **Panelisation:** the board is 54.5 mm on the short edge. Ask the fab whether
   it will be panelised; if so, confirm the mouse-bite or V-score positions and
   that no components sit within the breakaway zone.
8. **Silkscreen:** keep clear of pads; the reference designators are needed for
   hand rework.
9. **Edge clearance:** confirm no copper closer than 0.3 mm to the outline.

Deliver: Gerbers (RS-274X), Excellon drill file, and a fab drawing. Include the
board thickness and finish in the fab drawing rather than relying on file names.

---

## Assembly notes

The assembly is a straightforward single-sided SMT job plus a USB-C connector.

- **Sides:** all SMT parts on the top side. No bottom-side SMT, so no double
  reflow and no second stencil.
- **Joints:** roughly 220 – 260 SMT joints, dominated by passives.
- **Unique parts:** roughly 70 – 85 line items including passives.
- **Process:** lead-free reflow, IPC-A-610 Class 2 is sufficient for a prototype.
- **THT:** the headers and any through-hole parts can be hand-soldered or
  wave-soldered; specify which.
- **Turnkey vs consigned:** turnkey (the assembler sources the parts) is simpler
  and avoids shipping reels around, but you pay a markup and you inherit their
  stock risk. Consigned (you buy from a distributor and ship the parts) is
  cheaper on parts but shifts the logistics and any shortage onto you.

**Parts to check for stock before you commit:** the MAX-M10S and the ADXL372 are
the two expensive, long-lead items on this board. Ask for a stock date before
confirming a build date, and if either is out of stock, decide up front whether
you will accept a substitute — for the ADXL372, there is no drop-in equivalent
in the same footprint.

---

## Cost estimate

Prices below are ballpark figures for a small prototype run through a turnkey
PCBA service (PCBWay or equivalent). They are for budgeting only; the real
number comes from the vendor's generated quote.

### Two boards

| Line | Basis | Estimate |
| --- | --- | --- |
| PCB fabrication | Minimum order is typically 5 pcs; 10 pcs of this size is ~$5 | $5 – $10 |
| Assembly setup + stencil + labour | One-time setup dominates at this quantity | $80 – $120 |
| Components | 2 – 5 sets, see BOM | $100 – $150 |
| Shipping (DHL/FedEx) | ~1 kg | $40 – $60 |
| **Total** | | **$230 – $350** |
| Realistic outcome | Assemblers have a 5-piece minimum | **~$300 for 5 boards** |

### Ten boards

| Line | Estimate |
| --- | --- |
| PCB fabrication | $5 – $30 |
| Assembly (setup + labour) | $150 – $250 |
| Components (10 sets) | $500 – $750 |
| Shipping | $40 – $80 |
| **Total** | **$700 – $1,100** |
| **Unit cost** | **$70 – $110 per board** |

The important lesson from both tables: **at prototype quantities the PCB is
almost free and the money is in parts and assembly setup.** Unit cost falls
mainly by spreading the setup fee — at 50 pcs the setup cost per board drops by
roughly 5×, and component pricing improves another 15 – 30 %.

To get a vendor quote, fill in the PCBA form with approximately **250 SMT
joints** and **75 unique parts**, 2 layers, 54.5 × 86.23 mm, and turnkey
sourcing. Save the resulting PDF quote with its date so the number is traceable.

---

## Bring-up and test procedure

Do these in order. Do not skip steps — a fault found at step 9 costs you a whole
board, and a fault found at step 3 costs you a resistor.

### 1. Visual inspection, unpowered

- Check for solder bridges on the two LGA parts, the USB-C pins, and the fine
  passives. Use a microscope; the LGA pitch is too fine for naked-eye inspection.
- Confirm no components are missing or rotated, and that polarity marks match.
- Confirm no copper slivers or unplated edges.

### 2. Continuity, unpowered

- `+3V3` to `GND`: should read high resistance (tens of kΩ or more), not a short.
- `/USB_5V` to `GND`: same.
- `/BAT_PROT` to `/BAT_LOAD`: should read approximately the shunt value plus the
  protection path resistance.
- Every I²C and SPI line to `GND`: no shorts.

If `+3V3` reads a short, find it before applying power.

### 3. First power, current-limited

- Apply 5 V from a bench supply **with the current limit set to about 100 mA**.
- Watch the draw. A healthy idle board draws a few tens of mA; a board that
  slams into the limit has a short.
- Measure `+3V3` — should be 3.28 – 3.32 V.
- Measure `/GNSS_VCC` — should be present and close to 3.3 V.

### 4. USB enumeration

- Connect to a PC over USB-C. The ESP32-S3 should enumerate (native USB or a
  USB-serial bridge, depending on the module variant).
- Confirm 5 V is present on `/USB_5V` and the CC pull-downs are working — a
  USB-C-to-C cable from a USB-C host is the strict test; an A-to-C cable will
  work even with the CC resistors missing.

### 5. Module boot and programming

- Program a blink or serial-print sketch over USB.
- Confirm the module boots and the serial console works at the expected baud.
- Confirm the 2 × 5 header gives you access to the boot/reset lines if you use
  them.

### 6. I²C bus scan

- Scan the I²C bus and confirm every device answers at its expected address:
  BMP388, LSM6DSOX, PCF8563T, INA228.
- A missing device means a solder joint, a wrong address strap, or a
  non-populated part. A device answering at an unexpected address means an
  address pin is strapped differently than assumed.

### 7. Sensor sanity

- Read die temperature from each device — they should agree within a few degrees
  and match room temperature.
- Read the BMP388 pressure — near 1013 hPa at sea level, and it should change
  when you gently press or release near the port.
- Read the LSM6DSOX accelerometer — with the board flat, one axis reads about
  1 g; tilt the board and watch the vector rotate.
- Read the ADXL372 — confirm it responds on SPI and that `/ADXL_INT` toggles
  when you tap the board.
- Read the RTC — set a time, remove power for a minute, restore power, and
  confirm the time kept running.

### 8. Power monitor calibration

- With a known resistive load, compare the INA228 current reading against a
  multimeter in series.
- Confirm the sign convention, the shunt value in the configuration register,
  and the full-scale range. A wrong shunt value in firmware gives readings that
  are off by a constant factor — check it now, not later.

### 9. GNSS

- Go outdoors with a clear sky view. Cold start to first fix typically takes
  20 – 60 seconds. If you get no fix in five minutes outdoors, check the supply,
  the antenna connection and the UART/I²C configuration.
- Confirm the reported time matches reality to within a second.
- Compare the reported position to a known point.

### 10. Radio

- Confirm Wi-Fi association and BLE advertising.
- Check RSSI at a few metres. A weak or absent signal usually means metal or a
  battery over the antenna keep-out.

### 11. Thermal

- Run the board for ten minutes at your expected load and check the LDO and the
  module temperature by touch or IR. The LDO is the part that will get warm.

Record every measurement. A bring-up log with nominal values makes the second
board far easier to debug.

---

## Firmware interface

Firmware is not included in this repository; this section documents what the
hardware expects.

| Peripheral | Bus | Pins / nets | Notes |
| --- | --- | --- | --- |
| ESP32-S3 module | — | — | Native USB or bridge UART for programming |
| MAX-M10S | UART or I²C | `/GNSS_VCC` supply | Configure protocol at init |
| ADXL372 | SPI | `SPI_SCK`, `/ADXL_CS`, `/ADXL_INT` | Interrupt is the cheap way to detect impacts |
| LSM6DSOX | I²C | `I2C_SDA`, `I2C_SCL` | Optional interrupt pin if wired |
| BMP388 | I²C | `I2C_SDA`, `I2C_SCL` | Needs a conversion-time delay between reads |
| PCF8563T | I²C | `I2C_SDA`, `I2C_SCL` | Battery-backed; set once, read thereafter |
| INA228 | I²C | `I2C_SDA`, `I2C_SCL` | Configure shunt value and range before reading |
| PWM output | GPIO | `/PWM2` | General purpose |

Practical firmware advice:

- **Initialise I²C once, share the bus.** All four I²C devices live on the same
  two wires. Take a mutex or serialise access, or a long sensor read will
  collide with an RTC read and one of them will return garbage.
- **Use the ADXL372 FIFO**, not a polling loop. Polling a high-g part at high
  rate wakes the CPU constantly and burns the battery you are trying to measure.
- **Use GNSS time to discipline the RTC.** The RTC is accurate to tens of ppm;
  GNSS gives you absolute time. Re-sync on every fix.
- **Log the USB state with the current reading.** Otherwise your power data has
  an unexplained step in it wherever a cable was plugged in.
- **Deep sleep order matters.** Put the sensors to sleep first, then the GNSS,
  then the module. A sensor left in continuous-conversion mode will hold the
  rail up and defeat the point of sleeping.

---

## Known issues

These are open items on Rev B. Fix them before ordering a production batch.

1. **DRC is not clean.** The last full design-rule check reported unconnected
   pads in the double digits and several copper clearance violations. Unconnected
   pads mean the netlist and the layout disagree; on a two-layer board these are
   usually a forgotten via or a trace that stops one grid unit short. Run
   *Inspect → Clearance* and *Routing → Track Cleanup* and re-run DRC until it
   is empty.
2. **Solder-mask aperture bridge warnings on the LGA pads.** Expected and
   accepted: the apertures are intentionally 1:1 with the copper to prevent
   bridging under the body. Note this explicitly in the fab notes so the fab
   does not "fix" it.
3. **ERC power-output conflicts.** Multiple power-output pins on the same net
   (the regulator output pins and the GND pins on one device) plus a `PWR_FLAG`
   produce "power output and power output" errors. These are benign in KiCad
   when the pins genuinely are paralleled outputs; confirm each one rather than
   suppressing the class globally.
4. **ADXL372 land pattern provenance.** The footprint came from a
   vendor-supplied library (`PQFN50P322X300X114-16N`). Verify the pad
   dimensions and the pin-1 orientation against the ADXL372 datasheet land
   pattern before assembly — a mirrored LGA is not always visually obvious.
5. **No reverse-polarity protection on the battery connector** beyond the
   Schottky in the supply path. Confirm the connector keying physically prevents
   reverse insertion, or add a P-FET.
6. **Thermal margin of the LDO is unverified** in a sealed enclosure at full
   load. Measure before committing to a battery-only, enclosed deployment.
7. **USB-C is USB 2.0 only.** No USB-PD negotiation and no SuperSpeed. Do not
   design a charging scheme that assumes higher voltage.

---

## Design rules and constraints

The board was designed to clear the cheapest fabrication tier at common
low-cost fabs.

| Rule | Value used | Constraint |
| --- | --- | --- |
| Layers | 2 | 2-layer tier |
| Board size | 54.5 × 86.225 mm | under the 100 × 100 mm price tier |
| Min track | 0.1124 mm | above 0.10 mm |
| Min via drill | 0.20 mm | at the 0.20 mm standard minimum |
| Min via annular ring | standard | — |
| Mask expansion | 0.05 mm global, 0 mm on LGA | — |
| Board thickness | 1.6 mm | standard |
| Copper weight | 1 oz | standard |

Two-layer design consequences worth remembering: there is no solid ground plane
under the whole board, so return currents must follow their traces. Keep the
GNSS supply and the crystal traces short, keep the SPI clock away from the
crystal and the GNSS supply, and do not route signals across a plane split.

---

## Revision history

| Rev | Date | Change | Status |
| --- | --- | --- | --- |
| A | — | Initial layout | Superseded |
| B | 2026-10-29 | Two-layer rework, sensor set consolidated, LGA mask apertures set to 1:1 | DRC open, not yet built |

---

## File inventory

| File | Description |
| --- | --- |
| `StardancePCB.kicad_sch` | KiCad schematic (source of truth for the netlist) |
| `StardancB.kicad_pcb` | KiCad PCB layout, Rev B |
| `README.md` | This document |

When you generate the fabrication package, include the Gerbers, the Excellon
drill file, the BOM export, and the position (pick-and-place) file.

---

## Licence

Add your preferred licence here. For open hardware, CERN-OHL-P or CERN-OHL-S is
the usual choice; for a permissive hardware-plus-documentation release,
Apache-2.0 for the design files plus CC-BY-4.0 for the documentation also works.
