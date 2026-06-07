# Recovering extra accelerometer precision from the Core/Nunchuk buttons bytes

Notes from an investigation into whether wiiuse throws away real
accelerometer precision when it parses the button bytes. This investigation
became the `WITH_ACCEL_10BIT` feature and **is now implemented** - see
"Implementation" below for what actually shipped and how to enable it.

## Summary

The Wii Remote's accelerometer is a 10-bit (X) / 9-bit (Y) / 9-bit (Z)
sensor, and the Nunchuk's is a full 10-bit (X) / 10-bit (Y) / 10-bit (Z)
sensor, but the standard report format for each only carries **8 bits per
axis** (`msg[2]`, `msg[3]`, `msg[4]`) as a dedicated field. The missing
low-order bits are packed into otherwise-unused bits of the bytes that
precede (Wii Remote) or accompany (Nunchuk) the accelerometer bytes in every
report: the two Core Buttons bytes (`msg[0]`, `msg[1]`) for the Wii Remote,
the Nunchuk's own buttons byte (`msg[5]`) for the Nunchuk.

wiiuse did **not** destroy these bits. It just never read them:

- `wiiuse_pressed_buttons()` (`src/events.c`) masks a *local* 16-bit value
  with `WIIMOTE_BUTTON_ALL` (`0x1F9F`) before storing it in `wm->btns`.
  `WIIMOTE_BUTTON_ALL` deliberately excludes the 4 accel-LSB bits, so they
  vanish from the reported button state — but the underlying `msg` buffer is
  untouched by this.
- `handle_wm_accel()` (`src/events.c`) ran immediately afterward on the
  *same* `msg` buffer and, before this feature, only copied `msg[2..4]` into
  `wm->accel`. It never looked at `msg[0]`/`msg[1]`.
- `nunchuk_event()` (`src/nunchuk.c`) similarly only copied `msg[2..4]` into
  `nc->accel`, never looking at `msg[5]`'s upper bits (only its low 2 bits,
  the Z/C button state).
- Nothing in the transport layer (`os_nix.c`, `os_win.c`,
  `os_bt_embedded.c`) masks or rewrites the raw report before
  `propagate_event()` sees it either.

So the extra-precision bits were present and intact in `msg` all along —
they were simply discarded because nothing read them. Recovering them meant
adding extraction code at the two decode sites, not undoing any destructive
step.

## Bit mapping

### Wii Remote

Verified by reading Dolphin emulator source directly:
`Source/Core/Core/HW/WiimoteCommon/WiimoteReport.h` (the `ButtonData`
bit-field layout) and `DataReport.cpp`'s `GetAccelData()` (which
independently reimplements this and is validated against real hardware).
WiiBrew's wiki describes the same 10/9/9-bit split, but was not read
firsthand as part of this verification and is not cited as an
independently-confirmed source — see "Documentation status" below.

| bit            | mask   | meaning                        |
|----------------|--------|---------------------------------|
| `msg[0]` bit 5 | `0x20` | X accel, bit 0 of 2 extra LSBs |
| `msg[0]` bit 6 | `0x40` | X accel, bit 1 of 2 extra LSBs |
| `msg[1]` bit 5 | `0x20` | Y accel, extra LSB              |
| `msg[1]` bit 6 | `0x40` | Z accel, extra LSB              |

This only applies to the *standard* (non-interleaved) reports —
`WM_RPT_BTN_ACC`, `WM_RPT_BTN_ACC_EXP`, `WM_RPT_BTN_ACC_IR`,
`WM_RPT_BTN_ACC_IR_EXP` (report IDs `0x31`/`0x33`/`0x35`/`0x37`). These
are the only accelerometer-carrying reports `wiiuse_set_report_type()`
(`src/wiiuse.c`) ever requests, so this is safe for every mode wiiuse can
put the remote in. The Wii Remote also has a separate *interleaved*
reporting mode (`0x3e`/`0x3f`) where the Z axis is split across two
alternating reports in a different way entirely; wiiuse never selects it,
so it's not a live concern, but the extraction below would silently misparse
it if that mode were ever requested. `WITH_ACCEL_10BIT` does not attempt to
support it.

wiiuse defines masks for these 4 bits in `src/wiiuse.h`, now correctly named
(they were historically mislabeled):

```c
#define WIIMOTE_BUTTON_YACCEL_LSB  0x0020 /* msg[1] bit 5 -> Y LSB */
#define WIIMOTE_BUTTON_ZACCEL_LSB  0x0040 /* msg[1] bit 6 -> Z LSB */
#define WIIMOTE_BUTTON_XACCEL_LSB0 0x2000 /* msg[0] bit 5 -> X LSB bit 0 */
#define WIIMOTE_BUTTON_XACCEL_LSB1 0x4000 /* msg[0] bit 6 -> X LSB bit 1 */
```

### Nunchuk

Verified by reading Dolphin emulator source directly:
`Source/Core/Core/HW/WiimoteEmu/Extension/Nunchuk.h`'s `ButtonFormat`
bit-field (`acc_x_lsb`/`acc_y_lsb`/`acc_z_lsb`, each a 2-bit field), used as
`(msb << 2) | lsb` by the emulator's own `GetAccelX()`-style accessors.

| bits              | meaning                     |
|-------------------|-------------------------------|
| `msg[5]` bits 2-3 | X accel, 2 extra LSBs         |
| `msg[5]` bits 4-5 | Y accel, 2 extra LSBs         |
| `msg[5]` bits 6-7 | Z accel, 2 extra LSBs         |

Unlike the Wii Remote's lopsided 10/9/9 split, the Nunchuk gets full 10-bit
precision on every axis. `msg[5]` bits 0-1 remain the existing Z/C button
state (`NUNCHUK_BUTTON_Z`/`NUNCHUK_BUTTON_C`) and are unaffected.

## Extraction formula

Wii Remote (`handle_wm_accel()` in `src/events.c`):

```c
wm->accel.x = ((uint16_t)msg[2] << 2) | ((msg[0] >> 5) & 0x1) | (((msg[0] >> 6) & 0x1) << 1);
wm->accel.y = ((uint16_t)msg[3] << 1) | ((msg[1] >> 5) & 0x1);
wm->accel.z = ((uint16_t)msg[4] << 1) | ((msg[1] >> 6) & 0x1);
```

Nunchuk (`nunchuk_event()` in `src/nunchuk.c`):

```c
nc->accel.x = ((uint16_t)msg[2] << 2) | ((msg[5] >> 2) & 0x3);
nc->accel.y = ((uint16_t)msg[3] << 2) | ((msg[5] >> 4) & 0x3);
nc->accel.z = ((uint16_t)msg[4] << 2) | ((msg[5] >> 6) & 0x3);
```

## Implementation

This shipped as the opt-in `WITH_ACCEL_10BIT` feature. What actually landed,
in brief:

- A new CMake option, `WITH_ACCEL_10BIT` (default `OFF`), defines the
  compile-time macro `WIIUSE_ACCEL_10BIT` for the library build — the same
  `add_definitions()` mechanism `WITH_BT_EMBEDDED` already uses, and likewise
  **not** propagated to `wiiuse.pc`'s `Cflags`. An application that wants to
  interpret the widened field as extended precision must independently
  define `WIIUSE_ACCEL_10BIT` when it builds.
- Rather than adding new fields alongside the existing 8-bit ones (the
  approach originally sketched below in earlier drafts of this note), the
  existing `accel` field itself was widened: a new `vec3w_t` type
  (`uint16_t x, y, z`), and `wiimote_t.accel`, `wiimote_callback_data_t.accel`,
  and `nunchuk_t.accel` (which also serves as the Nunchuk's public
  callback-data mirror) all changed from `vec3b_t` to `vec3w_t` —
  **unconditionally**, regardless of the flag. With the flag off, the field
  holds exactly the same byte-range value it always has, just in a wider
  type; with it on, it holds the full extraction shown above. See the
  `vec3w_t` doc comment in `src/wiiuse.h` for the full details and caveats.
- `calculate_orientation()`/`calculate_gforce()` (`src/dynamics.c`, internal
  only) take the widened type and read it directly, with no narrowing —
  under `WIIUSE_ACCEL_10BIT`, their calibrated `orient`/`gforce` output now
  reflects the accelerometer's full extended precision instead of being
  silently collapsed back down to 8-bit-equivalent first. What moved instead
  is the calibration data (`accel_t.cal_zero`/`cal_g`, still byte-range):
  each caller's own per-axis shift left-shifts it up to the raw value's
  scale before the existing roll/pitch/gforce math runs unchanged. Because
  the Wii Remote (2/1/1 bits) and Nunchuk (2/2/2 bits) shift by different
  amounts, and the pre-existing MotionPlus-Nunchuk-passthrough path
  (`src/motion_plus.c`) never shifts at all, each caller passes its own
  shift amount explicitly (`WIIMOTE_ACCEL_*SHIFT` /
  `NUNCHUK_ACCEL_*SHIFT` / `NUNCHUK_PASSTHROUGH_ACCEL_*SHIFT` in
  `src/dynamics.h`, all `0` — a no-op — with the flag off) rather than the
  function assuming one fixed amount. With the flag off, or with the extra
  decoded bits all zero, output is bit-identical to before this change;
  with the flag on and the extra bits non-zero, `orient`/`gforce` measurably
  diverge in the expected direction and magnitude.
- `wiiuse_set_accel_threshold()`/`wiiuse_set_nunchuk_accel_threshold()`
  keep the same real-world sensitivity regardless of the flag: `events.c`'s
  threshold-crossing comparison rescales the (still plain, 8-bit-equivalent)
  threshold value by the same per-axis shift constant as the read path it's
  paired with — the Wii Remote's and direct Nunchuk's raw-accel comparisons,
  and the always-`0` MotionPlus-Nunchuk-passthrough comparison — instead of
  silently becoming more sensitive under the flag as it did before this
  change.
- Unit tests (`tests/test_accel_wiimote.c`, `tests/test_accel_nunchuk.c`,
  `tests/test_events_accel_threshold.c`, built via `check`/`ctest`, wired
  into CI) cover flag-off byte preservation, flag-on bit extraction against
  known synthetic report bytes including boundary patterns, calibrated
  orientation/gforce parity against an independent reference implementation
  (in both builds), calibrated output staying bit-identical when the extra
  bits are zero and diverging when they aren't, and accel_threshold
  crossing decisions scaled correctly per axis/path.
- `CHANGELOG.mkd` documents the breaking change to `accel`'s type
  (`uint8`→`uint16` per axis, unconditional, affecting every consumer's
  build regardless of the flag), and the calibrated-output/accel_threshold
  behavior changes under the flag.

Extending extended-precision handling to the Classic Controller, Guitar
Hero, Wii Balance Board, or other expansion devices remains out of scope
for this feature.

## Compatibility risks on real hardware

1. **Reverse-engineered, not an official spec.** Nintendo never
   published a protocol datasheet; this is community reverse engineering
   (WiiBrew) corroborated by an emulator project (Dolphin) that had to
   get it byte-exact against real controllers. Solid, but not a vendor
   guarantee.
2. **Clone/third-party Wiimotes and Nunchuks are common and often don't
   implement these bits faithfully**, since nothing in normal (non-extended)
   use reads them:
   - Bits hard-wired to `0` — harmless, just silently falls back to
     8-bit precision.
   - Bits carrying leftover/uninitialized garbage — the dangerous case:
     looks like valid extra precision but is actually noise, making the
     "higher precision" reading worse than the plain 8-bit value. Not
     distinguishable from the byte stream alone. This is why the feature is
     opt-in rather than always-on; there's no runtime detection or
     validation against genuine-vs-clone hardware.
3. **Genuine Nintendo hardware spans more than one accelerometer part**
   across the RVL-003 (original) and RVL-CNT-01-TR ("Wii Remote Plus",
   built-in Motion Plus) production runs. wiiuse already distinguishes
   these at connect time via Bluetooth CoD / HID product ID
   (`WIIUSE_WIIMOTE_REGULAR` vs `WIIUSE_WIIMOTE_MOTION_PLUS_INSIDE`, see
   `src/os_nix.c` and `src/os_win.c`). The *report byte layout*
   is identical on both, so no branching is needed to decode it — but
   the physical noise floor of the extra bits may differ between
   revisions even on genuine hardware, so the bits being present and
   correctly-decoded doesn't guarantee they're low-noise. Manual validation
   for this feature covered one genuine original Wii Remote (RVL-003) and
   one genuine Wii Remote Plus (RVL-CNT-01-TR); no clone hardware was
   available for testing.
4. **Report-mode safety net** — see interleaved-mode note above.

## Documentation status

- **Dolphin emulator source** — `Source/Core/Core/HW/WiimoteCommon/WiimoteReport.h`
  and `DataReport.cpp` (Wii Remote), `Source/Core/Core/HW/WiimoteEmu/Extension/Nunchuk.h`
  (Nunchuk) — fetched and read directly. These are the primary sources for
  both bit tables above.
- **WiiBrew wiki**, `Wiimote` article, "Core Buttons" section — widely
  cited as documenting the same Wii Remote 10/9/9-bit split ("X has 10 bits
  of precision, while Y and Z only have 9"), but was not read firsthand in
  any session that contributed to this feature (an earlier attempt hit an
  HTTP 403 on direct fetch, and a search-engine snippet / text-proxy
  re-read gave inconsistent bit numbers on a repeat query). It is not
  treated as an independently-verified source and is not cited directly in
  code comments; the Dolphin source above is the sole verified basis for
  the shipped implementation.
