# Wireless 65-75% Low-Profile Keyboard with ZMK

A complete reference for designing and building a custom Bluetooth keyboard from scratch — covering firmware, MCU selection, PCB design, switches, displays, trackpads, batteries, and cases.

---

## 1. Architecture Overview

Essentially a small embedded system. Here's how all the pieces relate:

```
┌─────────────────────────────────────────────────────────────┐
│  HOST (PC / Phone / Tablet)                                 │
│         ▲  Bluetooth 5.0 HID  /  USB-C HID                 │
├─────────┼───────────────────────────────────────────────────┤
│  MCU    │  nRF52840 (via nice!nano, SuperMini, or bare)     │
│         ├── ZMK Firmware (Zephyr RTOS)                      │
│         ├── Key Matrix (GPIO scan)                          │
│         ├── Display Driver (SPI / I2C)                      │
│         ├── Trackpad Driver (SPI / I2C — Cirque Pinnacle)   │
│         ├── Battery Monitoring (VDDH / ADC)                 │
│         └── Power Management (deep sleep ~20 µA)            │
├─────────────────────────────────────────────────────────────┤
│  HARDWARE                                                   │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌───────────┐  │
│  │ Key      │  │ Display   │  │ Trackpad │  │ Battery   │  │
│  │ Switches │  │ nice!view │  │ Cirque   │  │ LiPo      │  │
│  │ + Diodes │  │ or OLED   │  │ Pinnacle │  │ 3.7V      │  │
│  └──────────┘  └───────────┘  └──────────┘  └───────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. MCU / Controller Selection

ZMK runs on Zephyr RTOS, which supports 32/64-bit platforms. For wireless, the **Nordic nRF52840** is the dominant choice.

### Option A: Dev Board (Recommended for first build)

| Board | Price | Notes |
|-------|-------|-------|
| **nice!nano v2** | ~$25 | Gold standard. Pro Micro footprint. BQ24072 charger IC. Designed by a ZMK core team member. |
| **SuperMini nRF52840** | ~$5–8 | Budget alternative. Pin-compatible with nice!nano v2. Uses ceramic antenna. Works in ZMK as `nice_nano_v2`. |
| **Seeed XIAO nRF52840** | ~$10 | Smaller footprint. Built-in USB-C and battery management. Needs custom overlay for nice!view. |

Using a dev board means you don't need to design the radio, crystal, antenna matching network, USB-C, or charging circuit — the board handles all of that.

### Option B: Bare nRF52840 on Custom PCB (Advanced)

Fully integrated single-PCB design, Use a **module** like the Holyiot 18010 or E73-2G4M08S1C rather than the raw IC. The raw nRF52840 requires a 4-layer PCB with microvias to route all pins, which is impractical for most builders. A module integrates the chip, crystal, and antenna into a solderable package.

### Key nRF52840 Specs
- Bluetooth 5.0, ARM Cortex-M4F @ 64 MHz
- 1 MB Flash, 256 KB RAM
- Deep sleep: ~20 µA (nice!nano v2)
- USB 2.0 Full Speed built-in

---

## 3. Low-Profile Switches

**Critical:** Kailh Choc and Gateron KS-33 are NOT interchangeable. They use different stems, pin layouts, spacing, and keycaps. Choose one platform for your entire build.

### Kailh Choc V1 (PG1350)

- **Spacing:** 18 mm × 17 mm (Choc spacing — narrower than MX)
- **Travel:** 1.5 mm pre-travel, 3.0 mm total
- **Pins:** 2-pin + stabilizer legs
- **Keycap stem:** Proprietary rectangular
- **Pros:** Largest ecosystem of keycaps (MBK, CFX, CS, wrk.), most open-source PCB designs, thinnest option
- **Cons:** Non-standard spacing limits layout flexibility, fewer switch feel options than MX

### Recommendation for a Custom Build

---

## 4. Keycaps (Low-Profile)

For Kailh Choc V1 builds:

| Keycap Set | Profile | Material | Legends | Price Range |
|------------|---------|----------|---------|-------------|
| **MBK** | Low, flat, uniform | PBT | Dye-sub / blank | ~$25–40 |
| **ChocFox CFX** | Sculpted, cylindrical | PBT | Dye-sub | ~$25–35 |
| **wrk.** | Sculpted | PBT | Blank | ~$30 |
| **CS (Chicago Steno)** | Convex, ergonomic | PBT | Blank | ~$25 |

MBK is the most universally available and well-supported option.

---

## 5. Display Options

### nice!view (Recommended for Wireless)

- **Technology:** Sharp Memory-in-Pixel (LS011B7DH03) — looks like e-ink but with 30 Hz refresh
- **Resolution:** 160 × 68 pixels (1.08")
- **Power draw:** ~10 µA — over 1,000× less than OLED
- **Interface:** 3-wire SPI
- **Size:** 36 × 14 × 2.9 mm
- **Price:** ~$20
- **Firmware:** ZMK only
- **Shows:** Battery %, connection status, active layer, lock states, custom widgets/animations

Recommendation

**nice!view** is the clear winner for a wireless build. The power savings are enormous — the display barely registers on the power budget. It's what the nice!nano was designed to pair with.

### Display Wiring Notes

- nice!view uses SPI and needs one extra GPIO pin compared to OLED (CS pin)
- On boards with a standard OLED header (4-pin I2C), you'll need to bodge-wire the CS pin to an available GPIO
- Both the nice!view and the Cirque trackpad use SPI, so plan your pin allocation carefully — they can share MOSI/SCK but need separate CS pins

---

## 6. Trackpad / Pointing Device

### ZMK Pointing Device Support Status

Pointing device support is **officially merged into upstream ZMK**. This is no longer experimental. The firmware supports:

- Physical pointing devices (trackpads, trackballs, trackpoints)
- Mouse emulation via keybindings (`&mmv MOVE_UP`)
- Input processors for rotation correction, speed adjustment, scroll mode, etc.
- Split peripheral pointing devices (trackpad on one half of a split)

### Cirque Pinnacle GlidePoint (Recommended)

The most commonly used trackpad in custom keyboards. Available in circular form factors:

| Model | Size | Interface | Notes |
|-------|------|-----------|-------|
| **TM035035** | 35 mm diameter | SPI or I2C | Most common for keyboards |
| **TM023023** | 23 mm diameter | SPI or I2C | Smaller, fits tighter layouts |
| **TM040040** | 40 mm diameter | SPI or I2C | Larger touch area |

### Cirque Wiring (SPI — Recommended)

```
Cirque Pin    →  nRF52840 / nice!nano
─────────────────────────────────────
SDA/MOSI      →  SPI MOSI
SCL/SCK       →  SPI SCK
SS/CS         →  Any GPIO (chip select)
DR            →  Any GPIO (data ready, optional but recommended)
VCC           →  3.3V
GND           →  GND
```

**SPI vs. I2C selection:** Controlled by resistor R1 on the back of the Cirque module:
- R1 populated (470 KΩ) → SPI mode
- R1 removed → I2C mode (needs external 4.7 KΩ pull-ups on SDA/SCL)

**SPI is recommended** because it's faster, doesn't need pull-ups, and supports multiple trackpads via separate CS pins.

### ZMK Devicetree Configuration (SPI Example)

```dts
&spi0 {
    compatible = "nordic,nrf-spim";
    status = "okay";
    // pin definitions...

    glidepoint: glidepoint@0 {
        compatible = "cirque,pinnacle";
        reg = <0>;
        spi-max-frequency = <1000000>;
        status = "okay";
        dr-gpios = <&gpio0 XX GPIO_ACTIVE_HIGH>;
    };
};
```

### Other Trackpad Options

| Device | Driver Status | Notes |
|--------|---------------|-------|
| **Azoteq IQS5XX** | Community module | Larger rectangular trackpads |
| **Azoteq IQS7211E** | Community module | 30 mm circular, very low power (~1.5 mA), multi-touch |
| **PS/2 TrackPoint** | Community module | Classic ThinkPad-style nub |
| **PMW3610 / PAW3395** | Community modules | Trackball sensors |

---

## 7. Battery & Power

### Battery Selection

You need a **3.7V LiPo** (lithium polymer) cell. Key sizing considerations:

| Battery Code | Dimensions (mm) | Capacity | Use Case |
|-------------|-----------------|----------|----------|
| **301230** | 3 × 12 × 30 | ~110 mAh | Fits under socketed nice!nano. Minimal builds. |
| **401230** | 4 × 12 × 30 | ~130 mAh | Slightly thicker, more capacity |
| **502530** | 5 × 25 × 30 | ~300 mAh | Good balance for 75% build | <----
| **503450** | 5 × 34 × 50 | ~800 mAh | Large, for extended battery life |
| **603450** | 6 × 34 × 50 | ~1000 mAh | Maximum capacity if space allows |

### Charge Rate Rules

The nice!nano v2 charges at ~100 mA (BQ24072 IC). The battery should be sized so the charge rate falls between 0.25C and 1C:

- **Minimum battery:** 100 mAh (100 mA = 1C — max safe rate)
- **Ideal battery:** 200–500 mAh (0.5C–0.2C — gentle charging, long cycle life)
- **Don't go below 100 mAh** — overcharging shortens battery life dramatically

### Power Consumption Estimates

| State | Current Draw | Notes |
|-------|-------------|-------|
| Deep sleep | ~20 µA | After 15 min idle (configurable) |
| Active typing (BLE) | ~300–500 µA | Varies with typing speed |
| nice!view display | ~10 µA | Negligible |
| SSD1306 OLED | ~10,000+ µA | Avoid for wireless builds |
| Cirque trackpad | ~1–5 mA | Active use |
| RGB LEDs | 10,000–60,000+ µA | Avoid for wireless builds |

**ZMK Power Profiler:** Use https://zmk.dev/power-profiler to estimate battery life for your specific configuration.

### Estimated Battery Life (No OLED, No RGB)

| Battery | Light Use (2h/day) | Heavy Use (8h/day) |
|---------|-------------------|-------------------|
| 110 mAh | ~2 weeks | ~5–7 days |
| 300 mAh | ~5 weeks | ~2 weeks |
| 500 mAh | ~8 weeks | ~3.5 weeks |

### Charging Circuit (Custom PCB Only)

If you're using a nice!nano or SuperMini, the charging circuit is built in. For a custom PCB:

**TP4056** — Simple, cheap, widely available. BUT: don't use it as simultaneous charger + load driver. Add a Schottky diode (D2) to power the MCU directly from USB when plugged in, and a PMOSFET (Q1) as an ideal diode for battery operation.

**BQ24075** — Better option. "PowerPath" IC from TI that dynamically manages charging vs. load current. Has a SYSOFF pin for true power cutoff.

**Battery protection:** Add a **DW01A** IC for undervoltage/overvoltage cutoff, or use batteries with built-in protection circuits.

---

## 8. PCB Design

**Fully Integrated Custom PCB**
Design a PCB with an nRF52840 module (e.g., Holyiot 18010), USB-C connector, charging circuit, and everything integrated. This is what commercial wireless keyboards do. Requires solid electronics knowledge.

### PCB Design Tools

- **KiCad** (free, open-source) — the community standard for custom keyboards
- **marbastlib** — keyboard-focused KiCad library with switch footprints, MCU modules, etc.
- **Ergogen** — generates keyboard PCB outlines and switch positions from YAML config (great for ergonomic layouts)

### Key Matrix Design

For ~84 keys (75% layout), you'll wire switches in a matrix (rows × columns) with **one diode per switch** (1N4148 SMD or through-hole) to prevent ghosting.

A typical 75% matrix: 6 rows × 15 columns = 90 intersections (84 keys used).

Each row and column connects to a GPIO pin on the MCU. With a nice!nano (21 usable GPIOs), a 6×15 matrix uses 21 pins — tight but doable. Factor in pins for display (3 SPI + CS) and trackpad (shares SPI bus + own CS + DR), and you may need to use a matrix multiplexer or reduce the matrix.

### GPIO Pin Budget (nice!nano v2)

```
Available GPIOs:  ~21
Key matrix:       6 rows + 15 cols = 21   (uses all pins!)
Display (SPI):    MOSI, SCK, CS = 3       (shares SPI bus)
Trackpad (SPI):   CS, DR = 2              (shares MOSI/SCK with display)
─────────────────────────────────────────
Total needed:     ~26 pins

Problem: nice!nano doesn't have enough pins for all of this.
```

**Solutions:**
1. **Reduce matrix size** — use a smaller layout (65% instead of 75%), or use a more efficient matrix
2. **I/O expander** — add an MCP23017 (I2C, 16 GPIO) or 74HC595 shift register to expand pin count
3. **Duplex matrix** — scan rows bidirectionally to halve row pin count
4. **Use a bare nRF52840 module** — the Holyiot 18010 exposes 32 GPIOs, solving pin constraints entirely
5. **Charlieplexing** — more complex matrix encoding that reduces pin count

For a 75% with display AND trackpad, you'll likely need either an I/O expander or a bare module.

### Diodes

- **1N4148W** (SOD-123) — standard SMD diode for key matrix, ~$0.02 each
- **1N4148** (DO-35) — through-hole alternative
- Need one per key switch (84 for a 75%)

### Hot-Swap Sockets

- **Kailh Choc hot-swap sockets** — solder to PCB, allow switch removal without desoldering
- ~$0.15 each, need 84 for a 75%

### Connectors

- **USB-C**: Built into nice!nano. For custom PCB, use a mid-mount USB-C (e.g., GCT USB4085)
- **Battery connector**: JST PH 2.0mm 2-pin (standard for nice!nano)
- **Display header**: 5-pin for nice!view (VCC, GND, MOSI, SCK, CS)

### PCB Fabrication

Order from JLCPCB, PCBWay, or OSH Park. For a simple carrier board:
- 2-layer PCB, 1.6 mm thickness, FR4
- HASL or ENIG finish
- ~$5–15 for 5 boards (JLCPCB)
- Turnaround: ~1 week + shipping

### Reference Designs to Study

- **ebastler/zmk-designguide** — the go-to hardware design guide for ZMK keyboards
- **KBIC65** — 65% KiCad design with ProMicro footprint
- **TR-60** — 60% nRF52840 custom PCB, compatible with ZMK
- **zzeneg/stront** — split keyboard with Cirque trackpad + LCD display + ZMK

---

## 9. Case & Plate

### Plate Material

The plate sits between the switches and the PCB, providing mounting points:

| Material | Method | Pros | Cons |
|----------|--------|------|------|
| **FR4 (PCB material)** | Order as a second PCB | Cheap (~$5), precise, easy | Stiff, less premium feel |
| **3D-printed (PLA/PETG)** | FDM printer | Rapid prototyping, cheap | Flex, layer lines, less rigid |
| **3D-printed (Resin)** | SLA printer | Smooth finish, precise | Brittle, small build volume |
| **Aluminum** | CNC or laser cut | Premium, rigid | Expensive (~$50+), long lead time |
| **Acrylic** | Laser cut | Transparent options, cheap | Brittle, scratches |

For a first build, **FR4 is the sweet spot** — you can order it alongside your main PCB from the same fab house. Design it in KiCad as a second board with switch cutouts.

### Case Options

- **3D-printed case** — Most accessible. Design in Fusion 360 / FreeCAD. FDM works for prototyping; resin or SLS for a polished final product.
- **Sandwich case** — Top plate + PCB + bottom plate with standoffs. Simple, cheap, common in the community.
- **CNC aluminum** — Premium route. Services like SendCutSend or Xometry. Expect $100+ for a one-off.
- **Stacked acrylic** — Multiple laser-cut acrylic layers stacked and bolted. Distinctive look.

For low-profile Choc builds, the whole keyboard can be extremely thin (under 15 mm). Some builders skip the case entirely and just use a bottom plate + rubber feet.

---

## 10. ZMK Firmware Setup

### How ZMK Works

ZMK is built on Zephyr RTOS. You don't flash firmware like QMK — instead, you maintain a **zmk-config repository** on GitHub, and GitHub Actions compiles the firmware into a `.uf2` file you drag onto the MCU.

### Setup Steps

1. **Fork the ZMK config template** from https://github.com/zmkfirmware/unified-zmk-config-template
2. **Define your hardware** — create a "shield" (board definition) with:
   - `.overlay` file — devicetree describing your matrix, display, trackpad pin connections
   - `.keymap` file — your key layout, layers, combos, macros
   - `Kconfig.defconfig` — enable features (display, pointing, BLE)
3. **Push to GitHub** — Actions builds the firmware automatically
4. **Flash** — download the `.uf2` artifact, double-tap reset on the nice!nano, drag the file onto the USB mass storage device

### Key ZMK Features

- **Bluetooth multi-host** — connect to up to 5 devices, switch with key combos
- **Layers** — virtually unlimited, switch via momentary, toggle, or conditional
- **Combos** — press multiple keys simultaneously for a single output
- **Macros** — multi-key sequences on a single press
- **Mouse emulation** — cursor movement, clicks, and scroll from key bindings
- **Pointing device input processors** — transform trackpad input (rotate, scale, temporary layers)
- **Power management** — configurable idle timeout, deep sleep, external power control
- **Battery reporting** — percentage reported over BLE to host OS

### Keymap Editing Tools

- **nickcoutsos/keymap-editor** — web-based visual keymap editor for ZMK
- Manual `.keymap` file editing (devicetree syntax)

---

## 11. Full Bill of Materials (Estimated)

For a 75% low-profile wireless keyboard with display and trackpad, using a dev board approach:

| Component | Qty | Est. Cost | Source |
|-----------|-----|-----------|--------|
| nice!nano v2 (or SuperMini) | 1 | $8–25 | nicekeyboards.com, AliExpress |
| Kailh Choc V1 switches | 84 | $25–35 | chosfox.com, keeb.io |
| MBK or CFX keycaps | 1 set | $25–40 | chosfox.com, mkultra.click |
| Choc hot-swap sockets | 84 | $10–15 | keeb.io, chosfox.com |
| 1N4148W diodes (SOD-123) | 84 | $2–3 | LCSC, DigiKey |
| nice!view display | 1 | $20 | typeractive.xyz, nicekeyboards.com |
| Cirque TM035035 trackpad | 1 | $15–20 | mouser.com, digikey.com |
| LiPo battery (301230–503450) | 1 | $5–10 | AliExpress, Amazon |
| PCB fabrication (main + plate) | 5 ea | $10–20 | jlcpcb.com |
| Choc stabilizers (6.25u, 2u) | 1 set | $2–5 | chosfox.com |
| JST PH 2.0mm connectors | 2 | $1 | LCSC |
| Standoffs, screws, rubber feet | 1 kit | $5 | Amazon, AliExpress |
| 3D-printed case | 1 | $10–30 | Home printer or service |
| **Total estimate** | | **~$140–230** | |

---

## 12. Build Order / Workflow

Here's the recommended sequence from concept to working keyboard:

### Phase 1 — Design (2–6 weeks)
1. Finalize your layout (use keyboard-layout-editor.com)
2. Learn KiCad basics (ai03's PCB guide, then ebastler's ZMK design guide)
3. Design the schematic (key matrix, MCU footprint, connectors, display/trackpad headers)
4. Route the PCB
5. Design the plate (as a second KiCad project with switch cutouts)
6. Design or find a case (3D model)

### Phase 2 — Fabricate (1–3 weeks)
7. Order PCBs from JLCPCB / PCBWay
8. Order all components (switches, keycaps, diodes, sockets, controller, display, trackpad, battery)
9. 3D-print or order the case

### Phase 3 — Assemble (1–2 days)
10. Solder diodes to PCB
11. Solder hot-swap sockets
12. Solder/mount controller (socket headers recommended for nice!nano — makes it removable)
13. Connect display
14. Connect trackpad
15. Connect battery
16. Insert switches into plate + PCB
17. Assemble case

### Phase 4 — Firmware (1–2 days)
18. Create your zmk-config repo from the template
19. Write the shield files (.overlay, .keymap, Kconfig)
20. Build via GitHub Actions
21. Flash the .uf2 file
22. Test every key, layer, combo, display, and trackpad function
23. Iterate on keymap to taste

---

## 13. Key Resources

**ZMK Documentation**
- Official docs: https://zmk.dev/docs
- Power profiler: https://zmk.dev/power-profiler
- Pointing devices: https://zmk.dev/docs/features/pointing
- Display support: https://zmk.dev/docs/features/displays

**PCB Design**
- ebastler's ZMK hardware design guide: https://github.com/ebastler/zmk-designguide
- marbastlib (KiCad keyboard library): look for it in KiCad's plugin manager
- ai03's keyboard PCB guide (beginner): https://github.com/ruiqimao/keyboard-pcb-guide
- Ergogen (parametric layout generator): https://ergogen.xyz

**Reference Builds**
- KBIC65 (65%, ProMicro footprint): https://github.com/b-karl/KBIC65
- TR-60 (60%, bare nRF52840): https://github.com/hw-tinkerers/TR-60
- Stront (split, Cirque + display): https://github.com/zzeneg/stront

**Community**
- ZMK Discord (linked from zmk.dev)
- r/ErgoMechKeyboards (Reddit)
- kbd.news — keyboard news and switch database

---

## 14. Gotchas & Tips

- **Pin budget is tight on nice!nano.** A full 75% matrix + display + trackpad won't fit on 21 GPIOs. Plan for an I/O expander or a bare nRF52840 module.
- **SPI bus sharing works well.** The display and trackpad can share MOSI and SCK lines — just give each its own CS (chip select) pin.
- **Socket your nice!nano.** Use mill-max sockets or diode legs so you can remove/replace the controller without desoldering. This also creates a cavity underneath for the battery.
- **Don't use OLED for wireless.** The power draw will destroy your battery life. nice!view exists specifically to solve this.
- **Avoid RGB underglow/backlight on wireless.** LEDs draw 10–60+ mA, making wireless impractical.
- **Battery JST polarity varies.** There's no universal standard. Check your nice!nano's polarity marking before connecting a battery — reversed polarity will damage the board.
- **ZMK's Zephyr 4.1 update changed board naming.** `nice_nano_v2` is now just `nice_nano` (defaults to rev 2.0.0). Old configs may need updating.
- **Cirque R1 resistor matters.** If your trackpad isn't responding, check whether R1 is populated (SPI) or removed (I2C) to match your firmware config.
- **Use ZMK's power profiler** before finalizing your battery choice. It gives realistic estimates based on your exact configuration.
- **Order extra PCBs.** Minimum orders are usually 5 boards for ~$5 total. Having spares is invaluable when you find a design mistake.
