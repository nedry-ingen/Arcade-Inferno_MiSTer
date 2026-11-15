# Inferno + CRT Adjust

This fork adds [rmonic79/MiSTer-CRT-Adjust](https://github.com/rmonic79/MiSTer-CRT-Adjust)
(core-side variant, `crt_adjust.sv`) to the Inferno core. It lets you move and
resize the analog (15 kHz) picture from the OSD so it fits your CRT. `sys/` is
not modified.

## OSD — "CRT Adjust" page

| Option | Range | Status bits |
|---|---|---|
| CRT Frame H | 0, -4 … -24 px (left), +12, +8, +4 px (right). Works with CRT Adjust On or Off | 73:70 |
| CRT Adjust | Off / On | 101 |
| CRT H-Size | -16 … +15 (each step ≈ 1.6 %) | 100:96 |
| CRT H-Pos | -48 … +48 px, 2 px steps | 84:79 |
| CRT V-Shift | -16 … +15 lines | 78:74 |

- **CRT Frame H** shifts the HSync itself, so the whole frame, the picture and
  the OSD all move together. The left range stops at -24 px to leave a usable
  back porch (Inferno has 42 px of back porch).
- **H-Size / H-Pos / V-Shift** only show when CRT Adjust is On. With it Off the
  video is the untouched native signal. While it is On, HDMI follows the
  adjustment too (core-side variant).
- The module turns itself off when the scandoubler is in use (31 kHz output).
- Enlarging (H-Size > 0) also pushes the picture right; bring it back with a
  negative H-Pos.

## Files

| File | What |
|---|---|
| `rtl/crt_adjust.sv` | rmonic79's module. One fix: a `$signed()` so a negative H-Position is sign-extended in CONTENTSHIFT mode (without it any negative value blanks the picture). |
| `rtl/crt_adjust_glue.sv` | Glue (from `examples/core_side_snippet.v`): OSD decode, read clock-enable, `vb_in` for the next line, OSD-anchored DE, frame position |
| `Arcade-Inferno.sv` | `arcade_video` now feeds the glue, which drives `CE_PIXEL` and `VGA_*`; new OSD page; `status_menumask` bit 1 |
| `files.qip` | adds the two `.sv` files |
| `.github/workflows/build.yml` | builds the `.rbf` with Quartus on GitHub Actions and commits it to `releases/` |

Timing used by the glue: 6 MHz pixel from `clk_48` (8 clocks per pixel),
384 × 260 total, 282 px active → CONTENTSHIFT, read period (64 + H-Size)/8 clocks.

## Simulation check

An Icarus Verilog testbench using Inferno's timing checked that, with CRT
Adjust On, all 242 active lines come out with 282 pixels each and every pixel
matches the source (H-Size 0 and +8, H-Pos 0 and -16), and that CRT Frame H
moves the active video by exactly the selected number of pixels with CRT
Adjust On and Off.

## Installing

MRAs look for `inferno_*.rbf`. Copy `releases/Arcade-Inferno_YYYYMMDD.rbf` to
`_Arcade/cores/` as `Inferno_YYYYMMDD.rbf`, and remove the older Inferno .rbf.
