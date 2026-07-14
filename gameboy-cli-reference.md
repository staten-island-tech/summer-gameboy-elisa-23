# Game Boy Development CLI Reference

A reference guide for the RGBDS toolchain and BGB emulator command-line tools used in DMG (Game Boy) development.

---

## Overview

Building a Game Boy ROM from source involves three sequential steps, collectively called **compilation**:

1. **Assembly** — convert source code (`.rgbasm`) into object files (`.o`)
2. **Linkage** — combine object files into a single ROM (`.gb`)
3. **Fixing** — validate and patch the ROM header

---

## `rgbasm` — Assembler

Converts a `.rgbasm` source file into an object file containing DMG CPU machine code.

### Syntax

```
rgbasm [options] <source_file>
```

### Arguments

| Argument | Type | Description |
|---|---|---|
| `<source_file>` | Required | Path to the `.rgbasm` input source file |
| `-o <file>` | Required | Output object file path (conventionally `.o` extension) |
| `-Werror` | Flag | Treats all warnings as errors. Highly recommended — ignoring warnings will eventually cause bugs |
| `-Weverything` | Flag | Enables all warnings. Optional but improves code health and robustness. May produce false positives on valid code |
| `-Wall -Wextra` | Flag | Enables the most important warnings with a lower risk of false positives. A good alternative to `-Weverything` for personal projects |

### Example

```bash
rgbasm -Werror -Weverything -o main.o main.rgbasm
rgbasm -Werror -Weverything -o sample.o sample.rgbasm
```

### Notes

- Include files (`.rgbinc`) are **not** passed directly to `rgbasm`. Their contents are inserted into source files at compile time via the `include` directive.
- Run one `rgbasm` command per `.rgbasm` source file.

---

## `rgblink` — Linker

Takes one or more object files produced by `rgbasm` and links them into a single ROM file.

### Syntax

```
rgblink [options] <object_files...>
```

### Arguments

| Argument | Type | Description |
|---|---|---|
| `<object_files...>` | Required | One or more `.o` object files to link together |
| `-o <file>` | Required | Output ROM file path. Use `.gb` extension, as emulators expect this |
| `--dmg` | Flag | Disables VRAM and WRAM bank switching, which are CGB-only features. Use this for DMG-targeted ROMs |
| `--tiny` | Flag | Disables ROM bank switching. Only needed when ROM size exceeds 32KiB. Most programs in this book do not require it |
| `--map <file>` | Optional | Generates a **map file** listing the addresses of all sections, labels, and local labels. Also reports section sizes, making it easy to identify code that consumes the most space |
| `--sym <file>` | Optional | Generates a **symbol file** containing all label addresses per ROM bank. Emulators like BGB load this automatically to show label names in the debugger instead of raw addresses |

### Example

```bash
rgblink --dmg --tiny -o sample.gb main.o
rgblink --dmg --tiny --map sample.map --sym sample.sym -o base.gb main.o sample.o
```

### Notes

- The linker decides where to place sections in ROM unless an explicit address is given in the source (e.g. `rom0[$0100]`).
- BGB loads a `.sym` file automatically if it has the same base filename as the ROM and is in the same folder (e.g. `test.gb` → `test.sym`).
- Map and symbol files are purely for inspection and debugging — they have no effect on ROM behaviour.

---

## `rgbfix` — ROM Fixer

Finalises the ROM by writing correct values into the cartridge header and validating checksums. The ROM will not pass the DMG boot checks without this step.

### Syntax

```
rgbfix [options] <rom_file>
```

### Arguments

| Argument | Type | Description |
|---|---|---|
| `<rom_file>` | Required | Path to the `.gb` ROM file to fix. **The file is modified in place (overwritten)** |
| `--title <name>` | Optional | Sets the game title in the cartridge header. Must be **11 characters or fewer** — only 11 bytes are reserved for the title |
| `--pad-value <byte>` | Optional | Pads unused ROM regions with the specified byte value. Using `0` is recommended: zero is the opcode for `nop`, making unused regions easy to identify in the debugger |
| `--validate` | Flag | Fixes the Nintendo logo data and both header checksums (header checksum and global checksum). The DMG boot ROM verifies these before executing any code — the ROM will not run if they are invalid |

### Example

```bash
rgbfix --title game --pad-value 0 --validate sample.gb
```

### Notes

- `rgbfix` must be the **last** step in the build process.
- The `--validate` flag is essential. Without it, the DMG will refuse to run the ROM.
- This step is independent of the number of source files used — the command is always the same regardless of project size.

---

## `rgbgfx` — Graphics Converter

Converts PNG image files to DMG-compatible graphics formats: CHR files (tile data) and TLM files (tilemap indices).

### Syntax

```
rgbgfx [options] <input_png>
```

### Arguments

| Argument | Type | Description |
|---|---|---|
| `<input_png>` | Required | Input PNG file. Should contain only up to four colours |
| `--output <file>` | Required | Output CHR file containing the converted tile data |
| `--unique-tiles` | Flag | When converting tilemaps, generates a CHR file with only the unique tiles needed — no duplicates or unused tiles |
| `--tilemap <file>` | Optional | Output TLM file containing tile indices referencing tiles in the CHR file. Used when converting tilemap images |

### Examples

**Convert a tileset (tiles only):**
```bash
rgbgfx --output tileset.chr tileset.png
```

**Convert a tilemap (produces both a CHR and a TLM file):**
```bash
rgbgfx --unique-tiles --output tileset.chr --tilemap tilemap.tlm tilemap.png
```

### Notes

- A **CHR file** is a collection of tile data in DMG tile format (16 bytes per 8×8 tile).
- A **TLM file** is a list of 1-byte tile indices (1,024 bytes for a full 32×32 tilemap).
- `rgbgfx` cannot convert multiple tilemaps to a shared CHR file in one pass. For that use case, a custom tool is needed.
- Full documentation: [https://rgbds.gbdev.io/docs/master/rgbgfx.1](https://rgbds.gbdev.io/docs/master/rgbgfx.1)

---

## `gfxconv` — Custom Graphics Converter

A custom PNG conversion tool (provided with this book's repository) that converts a tileset and any number of tilemap PNGs to CHR and TLM files. Unlike `rgbgfx`, it preserves tileset organisation so that multiple tilemaps can share one CHR file.

### Syntax

```
gfxconv <tileset_png> [tilemap_png...]
```

### Arguments

| Argument | Type | Description |
|---|---|---|
| `<tileset_png>` | Required | Input tileset PNG. Must contain exactly 384 8×8 tiles (a 128×192 px image). The output CHR file takes the same base name |
| `[tilemap_png...]` | Optional | One or more tilemap PNG files (256×256 px each). A TLM file is generated for each one. If none are provided, no TLM files are generated |

### Example

```bash
gfxconv tileset.png background.png window.png
```

### Notes

- The tool makes specific assumptions about PNG dimensions. See `README.md` in the repository for full details: [https://github.com/mdagois/gca/blob/main/README.md](https://github.com/mdagois/gca/blob/main/README.md)
- Source code is available in the same repository and can serve as a reference for building custom conversion tools.

---

## `bgb` — BGB Emulator

Runs a Game Boy ROM in the BGB emulator. BGB also includes a full debugger accessible at any time during execution.

### Syntax

```
bgb [options]
```

### Arguments

| Argument | Type | Description |
|---|---|---|
| `--rom <file>` | Optional | Loads the specified `.gb` ROM file on launch |
| `--watch` | Optional | Enables **watch mode**: BGB continuously monitors the ROM file for changes and automatically reloads it whenever it is recompiled. Breakpoints are preserved across reloads. Highly recommended during active debugging sessions |

### Examples

```bash
# Launch with a ROM
bgb --rom sample.gb

# Launch with watch mode enabled (auto-reload on recompile)
bgb --rom sample.gb --watch
```

### Notes

- On 64-bit systems the executable may be named `bgb64`.
- BGB automatically loads a `.sym` symbol file if it shares the same base name as the ROM and is in the same folder.
- The debugger is opened at any time by pressing `ESC`.
- Full manual: [https://bgb.bircd.org/manual.html](https://bgb.bircd.org/manual.html)

---

## Joypad Input

The DMG has eight keys: UP, DOWN, LEFT, RIGHT, A, B, START, and SELECT. Input is read through the `rP1` I/O register, four keys at a time.

### rP1 Register Layout

| Bits | Role |
|---|---|
| 7–6 | Unused |
| 5–4 | Selector — choose which group of keys to poll |
| 3–0 | Poll result (bit = 0 means key is held) |

### Selector Values

| Constant | Polls |
|---|---|
| `P1F_GET_DPAD` | D-pad (UP, DOWN, LEFT, RIGHT) |
| `P1F_GET_BTN` | Buttons (A, B, START, SELECT) |
| `P1F_GET_NONE` | Disables polling (write after reading) |

### Input Flags and Bits (`hardware.rgbinc`)

| Flag constant | Bit constant | Key |
|---|---|---|
| `PADF_DOWN` ($80) | `PADB_DOWN` (7) | DOWN |
| `PADF_UP` ($40) | `PADB_UP` (6) | UP |
| `PADF_LEFT` ($20) | `PADB_LEFT` (5) | LEFT |
| `PADF_RIGHT` ($10) | `PADB_RIGHT` (4) | RIGHT |
| `PADF_START` ($08) | `PADB_START` (3) | START |
| `PADF_SELECT` ($04) | `PADB_SELECT` (2) | SELECT |
| `PADF_B` ($02) | `PADB_B` (1) | B |
| `PADF_A` ($01) | `PADB_A` (0) | A |

Use `PADF_*` flags with the `and` instruction (works on `a` only, can test multiple keys at once). Use `PADB_*` bits with the `bit` instruction (works on any 8-bit register, one key at a time). A raised zero flag means the key is active.

### Polling Sequence

Reading the d-pad requires **2 `ldh` reads** (first is a stabilisation wait). Reading buttons requires **6 `ldh` reads** (first five are stabilisation waits). Always write `P1F_GET_NONE` to `rP1` after reading to disable polling.

### Input Structure (`utils.rgbinc`)

The recommended pattern stores all input state in a 4-byte WRAM struct:

```
rsreset
def PAD_INPUT_CURRENT  rb 1   ; latest polled state
def PAD_INPUT_PREVIOUS rb 1   ; state from previous frame
def PAD_INPUT_PRESSED  rb 1   ; keys that transitioned held → not-held this frame
def PAD_INPUT_RELEASED rb 1   ; keys that transitioned not-held → held this frame
def sizeof_PAD_INPUT   rb 0
```

In all fields, a bit value of **0 means the key is active**; **1 means inactive**. Initialise all fields to `$FF`.

### Input Macros

| Macro | Description |
|---|---|
| `InitPadInput <struct>` | Initialises all struct fields to `$FF` |
| `UpdatePadInput <struct>` | Polls d-pad and buttons, updates all four fields |
| `TestPadInput_HeldAll <struct>, <mask>` | Raises zero flag if all masked keys are **held** |
| `TestPadInput_Pressed <struct>, <mask>` | Raises zero flag if masked keys were **pressed** this frame |
| `TestPadInput_Released <struct>, <mask>` | Raises zero flag if masked keys were **released** this frame |

### Usage Notes

- Call `UpdatePadInput` **once per frame** in the main loop. If needed in multiple places, wrap it in a function.
- Call `InitPadInput` **once** during initialisation.
- Use `TestPadInput_HeldAll` for continuous actions (e.g. scrolling). Use `TestPadInput_Pressed` or `TestPadInput_Released` for one-shot actions (e.g. toggling a window) to avoid repeated triggering while a button is held.

### Joypad Interrupt

The joypad interrupt (flag `IEF_HILO`, handler at `$0060`) fires when any selected key changes state. **Not recommended for general input handling** due to three major drawbacks:

1. Can trigger at any point in the frame, conflicting with the main loop
2. May fire multiple times per physical press due to button bounce
3. Cannot distinguish between keys sharing the same bit (A shares with RIGHT, B with LEFT, SELECT with UP, START with DOWN) when all eight keys can generate interrupts

Use polling instead. The interrupt is useful only for waking the CPU from a low-power `halt` state, and even then requires a vblank flag (`WRAM_IS_VBLANK`) to verify the correct interrupt fired before proceeding.

---

## Memory Variables with `rsset` / `rsreset`

`rsset` and `rsreset` declare contiguous memory offsets without manual arithmetic. They are the preferred way to lay out WRAM variables and data structures.

### Syntax

```
rsset <start_address>    ; set counter to address (e.g. rsset _RAM for WRAM)
rsreset                  ; equivalent to rsset 0 (used for struct definitions)

def <NAME> rb <n>        ; reserve n bytes, increment counter by n
def <NAME> rw <n>        ; reserve n words (2 bytes each)
def <NAME> rl <n>        ; reserve n longs (4 bytes each)
def sizeof_<STRUCT> rb 0 ; captures current counter without advancing it
```

### Example

```
rsset _RAM
def WRAM_PAD_INPUT   rb sizeof_PAD_INPUT
def WRAM_BG_SCX      rb 1
def WRAM_BG_SCY      rb 1
def WRAM_END         rb 0

def WRAM_USAGE equ (WRAM_END - _RAM)
println "WRAM usage: {d:WRAM_USAGE} bytes"
assert WRAM_USAGE <= $2000, "Too many bytes used in WRAM"
```

Adding a `sizeof_` constant at the end of each struct records the total size and allows safe reservation with `rb sizeof_<STRUCT>`. Use `println` and `assert` to track and enforce WRAM budgets at compile time.

---

## DMG Audio

The DMG has four audio channels mixed to two output terminals (left and right). All channel registers follow the naming pattern `rNRxy`, where `x` is the channel number (1–4) and `y` is the register index (0–4).

### Sound Hardware Control Registers

| Register | Bits | Description |
|---|---|---|
| `rNR50` | 6–4 | Left terminal volume (0–7) |
| `rNR50` | 2–0 | Right terminal volume (0–7) |
| `rNR50` | 7, 3 | Enable Vin on left / right terminal |
| `rNR51` | 7–4 | Route channels 4–1 to left terminal |
| `rNR51` | 3–0 | Route channels 4–1 to right terminal |
| `rNR52` | 7 | Enable sound hardware (0 = off, saves ~15% battery) |
| `rNR52` | 3–0 | Read-only flags: channel 4–1 currently playing |

**Always enable the sound hardware via `rNR52` bit 7 before writing to any other sound register.** Writes to channel registers are ignored while the hardware is disabled.

### Initialisation Example

```
copy [rNR52], AUDENA_ON   ; enable sound hardware
copy [rNR50], $77         ; max volume on both terminals
copy [rNR51], $FF         ; all channels to both terminals
```

### Frequency Formulas (Channels 1, 2, and 3)

Frequency is encoded as an 11-bit value split across two registers (low 8 bits in `rNRx3`, high 3 bits in `rNRx4` bits 2–0).

| Channel | Hz from value | Value from Hz |
|---|---|---|
| 1 & 2 | `131072 / (2048 - value)` | `2048 - (131072 / freq)` |
| 3 | `65536 / (2048 - value)` | `2048 - (65536 / freq)` |

Round fractional values down. Always set `rNRx4` last — sound starts the moment the restart flag (bit 7) is written.

---

### Channel 1 — Quadrangular Wave with Sweep

| Register | Bits | Access | Restart needed | Usage |
|---|---|---|---|---|
| `rNR10` | 6–4 | R/W | — | Sweep step length (0 = disabled) |
| `rNR10` | 3 | R/W | — | Sweep direction (0 = add, 1 = sub) |
| `rNR10` | 2–0 | R/W | — | Sweep shift number |
| `rNR11` | 7–6 | R/W | — | Duty cycle |
| `rNR11` | 5–0 | W | — | Sound length (5-bit) |
| `rNR12` | 7–4 | R/W | Yes | Volume (0–15) |
| `rNR12` | 3 | R/W | Yes | Envelope direction (0 = down, 1 = up) |
| `rNR12` | 2–0 | R/W | Yes | Envelope step length (0 = disabled) |
| `rNR13` | 7–0 | W | — | Frequency low 8 bits |
| `rNR14` | 7 | W | — | Restart sound flag |
| `rNR14` | 6 | R/W | — | Length counter enable |
| `rNR14` | 2–0 | W | — | Frequency high 3 bits |

**Duty cycles** (`rNR11` bits 7–6): `$00` = 12.5%, `$40` = 25%, `$80` = 50% (square), `$C0` = 75%. Note: 25% and 75% sound identical.

**Sound length** (`rNR11` bits 5–0): `length_seconds = (64 - value) / 256`. Only active when `rNR14` bit 6 is set. Range: 3.9 ms (value 63) to 250 ms (value 0).

**Volume envelope step length** (`rNR12` bits 2–0): `step_seconds = value / 64`. Range: 15.6 ms (1) to 109.4 ms (7).

**Frequency sweep step length** (`rNR10` bits 6–4): `step_seconds = value / 128`. Range: 7.6 ms (1) to 54.7 ms (7).

**Sweep formula (addition):** `freq(n) = freq(n-1) + freq(n-1) / (1 << shift)`. Sound stops if frequency overflows 11 bits. Use subtraction (`rNR10` bit 3 = 1) to lower pitch over time.

---

### Channel 2 — Quadrangular Wave (no sweep)

Identical to Channel 1 except no sweep function and no `rNR20` register. Registers: `rNR21`–`rNR24` with the same bit layout as Channel 1's `rNR11`–`rNR14`.

---

### Channel 3 — Free-Form Wave

Plays a custom 32-sample waveform stored in **wave RAM** (`$FF30`–`$FF3F`, 16 bytes). Each byte holds two 4-bit samples (high nibble first). No envelope or sweep.

| Register | Bits | Access | Usage |
|---|---|---|---|
| `rNR30` | 7 | R/W | Sound enable (0 = off, 1 = on) |
| `rNR31` | 7–0 | R/W | Sound length (8-bit): `(256 - value) / 256` seconds |
| `rNR32` | 6–5 | R/W | Volume: `%00`=0%, `%01`=100%, `%10`=50%, `%11`=25% |
| `rNR33` | 7–0 | W | Frequency low 8 bits |
| `rNR34` | 7 | W | Restart sound flag |
| `rNR34` | 6 | R/W | Length counter enable |
| `rNR34` | 2–0 | W | Frequency high 3 bits |

**Always clear `rNR30` (sound enable) before writing to wave RAM** — behaviour during playback is undefined and may corrupt the waveform. Re-enable and then set the restart flag to start playback.

**Common waveforms (wave RAM hex values):**

| Waveform | Bytes |
|---|---|
| Square 50% | `$FF $FF $FF $FF $FF $FF $FF $FF` / `$00 $00 $00 $00 $00 $00 $00 $00` |
| Triangle | `$01 $23 $45 $67 $89 $AB $CD $EF` / `$ED $CB $A9 $87 $65 $43 $21 $00` |
| Sawtooth | `$00 $11 $22 $33 $44 $55 $66 $77` / `$88 $99 $AA $BB $CC $DD $EE $FF` |
| Sine | `$89 $AC $DE $EF $FF $EE $DC $A9` / `$86 $53 $21 $10 $00 $12 $23 $56` |

Duty cycle can be fine-tuned in increments of 3.125% by adjusting the ratio of `$F` to `$0` samples. Volume can be varied beyond the four presets by scaling sample values directly in the waveform data.

---

### Channel 4 — Noise

Produces pseudorandom noise via a **linear-feedback shift register (LFSR)**.

| Register | Bits | Access | Restart needed | Usage |
|---|---|---|---|---|
| `rNR41` | 5–0 | W | — | Sound length |
| `rNR42` | 7–4 | R/W | Yes | Volume (0–15) |
| `rNR42` | 3 | R/W | Yes | Envelope direction (0 = down, 1 = up) |
| `rNR42` | 2–0 | R/W | Yes | Envelope step length |
| `rNR43` | 7–4 | R/W | — | Prescaler shift count (values 14–15 invalid) |
| `rNR43` | 3 | R/W | — | LFSR mode: 0 = 15-bit (white noise), 1 = 7-bit (tonal patterns) |
| `rNR43` | 2–0 | R/W | — | Clock divider (0 = divisor 0.5, 1–7 = divisor value) |
| `rNR44` | 7 | W | — | Restart sound flag |
| `rNR44` | 6 | R/W | — | Length counter enable |

**Counter update frequency:** `524288 / clock_divisor / prescaler_divisor`

**15-bit mode** (32,767 values) produces white noise. **7-bit mode** (127 values) produces more tonal, patterned noise and can approximate musical notes.

**Example sound effect register values:**

| `rNR41` | `rNR42` | `rNR43` | `rNR44` | Sound |
|---|---|---|---|---|
| `$00` | `$F3` | `$27` | `$80` | Object breaking |
| `$00` | `$F4` | `$46` | `$80` | Explosion |
| `$00` | `$0F` | `$57` | `$80` | Spaceship propulsion |
| `$00` | `$F3` | `$1F` | `$80` | Power-up (7-bit mode) |

---

### General Audio Tips

- **Always set `rNRx4` last.** Sound starts the moment the restart flag is written; other registers must be configured first to avoid sonic artefacts.
- **Changes to `rNR12` / `rNR22` do not take effect until restart.** Buffer the value in WRAM and write it as close as possible to setting the restart flag.
- **Do not assume register values during playback.** Envelope and sweep functions modify them over time.
- You can modify frequency registers mid-playback without restarting the sound (useful for playing melodies on one channel). Do not modify volume-related registers this way.
- To pan sound: set both `rNR51` bits for a channel to centre, one bit for hard left/right pan.

---

## Full Build Example

A complete build sequence for a two-file project:

```bash
# 1. Assemble each source file into an object file
rgbasm -Werror -Weverything -o main.o main.rgbasm
rgbasm -Werror -Weverything -o sample.o sample.rgbasm

# 2. Link object files into a ROM (with map and symbol files)
rgblink --dmg --tiny --map sample.map --sym sample.sym -o base.gb main.o sample.o

# 3. Fix the ROM header and checksums
rgbfix --title game --pad-value 0 --validate base.gb

# 4. Run in BGB with auto-reload on recompile
bgb --rom base.gb --watch
```

---

## Debugger Keyboard Shortcuts (BGB)

| Shortcut | Action | Applicable View |
|---|---|---|
| `ESC` | Open / close debugger | — |
| `F5` | Open VRAM viewer | — |
| `CTRL-G` | Go to address | All except register |
| `CTRL-A` | Go to current PC | Code |
| `F2` | Toggle breakpoint | Code |
| `F3` | Step over (skip into functions) | Code |
| `F7` | Trace (step into functions) | Code |
| `F8` | Step out of current function | Code |
| `F9` | Run until next breakpoint | Code |