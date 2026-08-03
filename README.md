# optra-ac-display

Replacement display + control interface for a dead Chevrolet Optra Digital AC control panel LCD.

## Background

The original AC control panel used a monochrome, calculator-style LCD driven by a dedicated
controller IC. That IC failed (shorted), and no direct replacement was available. Rather than
sourcing a used panel, this project intercepts the serial data that the car's AC main controller
board sends to the (now-dead) LCD driver IC, decodes it, and re-renders the same information on an
OLED display using an ESP32.

In short: the AC controller thinks it's still talking to the original LCD. It has no idea an ESP32
is quietly listening in and drawing everything on an OLED instead.

## How it works

1. **Signal interception** — The ribbon cable that originally ran from the AC main controller board
   to the LCD driver IC is intercepted and routed through the ESP32 instead. Three signals are
   captured: `CLK`, `DATA`, and `EN` (enable/latch).
2. **Shift register decode** — The protocol turned out to be a serial shift-register style
   interface. On each falling edge of `CLK`, the ESP32 (via interrupt) shifts in one bit from
   `DATA`. A full frame is 64 bits, framed by `EN` going high (start) and low (end).
3. **Frame decode** — Once a complete 64-bit frame is captured, it's decoded into a `FrameInfo`
   struct: fan speed, AC on/off, auto mode, recirculation/fresh air, vent mode (face/feet/defrost),
   and the 7-segment temperature digits — all reverse-engineered bit-by-bit from the original
   signal.
4. **OLED rendering** — The decoded state is rendered on a 256×64 SSD1322 OLED (via U8g2),
   reproducing the original panel's icons and layout using hand-drawn XBM bitmaps.
5. **Encoder passthrough** — The original panel's rotary encoders (temp, fan) are read by the
   ESP32 and "replayed" out to the AC main controller board via simulated quadrature signals, so
   turning the physical knobs still works exactly as before.
6. **Button passthrough** — Physical buttons (AUTO, AC, MODE, OFF, front defrost) are simulated via
   open-drain GPIO outputs wired into the original button circuit, so they can be triggered
   programmatically as well as physically.
7. **Bluetooth control (extra feature)** — A classic Bluetooth Serial (SPP) interface lets commands
   be sent from a phone to trigger encoder turns and button presses remotely, as an experimental
   add-on beyond the original hardware.

## Hardware

- ESP32 dev board
- SSD1322-based 256×64 OLED (4-wire SPI)
- Rotary encoders (temp, fan) — intercepted from the original panel
- Original AC panel's physical buttons (AUTO, AC, MODE, OFF, front defrost)
- Direct tap into the ribbon cable between the AC main controller and the original (dead) LCD
  driver IC

## Status

Functional daily-driver replacement for the dead LCD. Known issue: rare display glitches
(phantom/incorrect icons appearing momentarily) have been observed, correlated with an active
Bluetooth connection — believed to be RF-induced noise on the intercepted signal lines corrupting
occasional frame captures. Mitigations in progress: frame-confirmation logic (require two
consecutive matching captures before updating the display) and enable-line debounce.

## Disclaimer

This project involves tapping into and interpreting undocumented signals from real vehicle
electronics via reverse engineering. Wiring and bit-mapping are specific to this Optra AC panel
and were derived experimentally — they are not guaranteed to be correct or safe for other vehicles
or panel revisions. Use at your own risk.
