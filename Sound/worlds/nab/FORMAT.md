# BF2 Sound Source Format — nab/ walkthrough

## Directory contents

| File | Role |
|---|---|
| `nab.req` | Top-level container spec → drives `LevelPack.exe` to produce `nab.lvl` |
| `nab2gcw.req` | Container spec for the GCW (Imperial/Rebel) bank level → `nab2gcw.lvl` |
| `nab2cw.req` | Container spec for the CW (Republic/CIS) bank level → `nab2cw.lvl` |
| `nab2.stm` | Stream manifest for PS2/Xbox mono ambient → contributes to `nab.str` |
| `nab2.st4` | Stream manifest for Xbox/PC 4-channel (stereo fnt+bck) ambient |
| `nab2_emt.stm` | Stream manifest for ambient emitter stream |
| `nab2gcw.sfx` | Sample bank source for GCW → compiled to `nab2gcw.bnk` |
| `nab2cw.sfx` | Sample bank source for CW → compiled to `nab2cw.bnk` |
| `nab2.snd` | Sound property configs for ambient emitters → compiled to `nab2.config` |
| `nabcw_music_config.snd` | Music stream event definitions → `nabcw_music_config.config` |
| `nabgcw_music_config.snd` | Music stream event definitions → `nabgcw_music_config.config` |
| `nabcw_music.mus` | Music state machine (fades, timing, priority) → compiled to `.config` |
| `nabgcw_music.mus` | Same for GCW era |
| `nab2cw_foley.ffx` | Foley FX group mappings (terrain type → sound name) |
| `nab2gcw_foley.ffx` | Same for GCW era |
| `nab_objective_vo.snd` | Objective VO sound events (cross-era) |

---

## Build tools

| Tool | Input | Output |
|---|---|---|
| `SoundFLMunge.exe` | `.sfx` | `.bnk` (binary sample bank, one per platform) |
| `configmunge.exe` | `.snd`, `.mus`, `.ffx` | `.config` (binary event/property tables) |
| `StreamPack.exe` (or similar) | `.stm`, `.st4` | audio data packed into `.str` stream file |
| `LevelPack.exe` | `.req` | `.lvl` / `.str` — bundles everything listed in the req |

---

## nab.req — top-level container spec

```
ucft
{
    REQN { "str"  "align=2048"  "nab2_emt"  "nab2" }
    REQN { "lvl"  "nab2gcw"  "nab2cw" }
}
```

- Each `REQN` block: first string = output file extension, rest = asset names.
- `align=2048` — stream data must be 2048-byte block aligned (required for optical-drive streaming).
- **Result:** `LevelPack.exe` produces `nab.str` (stream audio) and `nab2gcw.lvl` + `nab2cw.lvl` (sample banks).

> Note: the final shipped `nab.lvl` is likely a further top-level bundle that embeds both the `.str`
> and the `.lvl` bank files, or `nab.lvl` refers to just one of them depending on the platform.

---

## nab2gcw.req / nab2cw.req — bank level container spec

```
ucft
{
    REQN { "bnk"  "align=2048"  "nab2cw" }     // the compiled sample bank
    REQN { "config"
           "global_world"        // shared global sound event defs
           "nabcw_music_config"  // music stream event defs
           "nabcw_music"         // music state machine
           "nab2"                // ambient emitter sound props
           "nab2cw_foley"        // foley terrain mappings
           ...                   // unit VO, foley, vehicle configs
    }
}
```

- `LevelPack.exe` resolves each name by extension: looks for `nab2cw.bnk`, `global_world.config`, etc.
- The result is a UCF `.lvl` file containing a sample bank chunk (0x0fb40705 info + 0xd872e2a5 data)
  plus all the config chunks.

---

## .sfx — sample bank source

This is the most important file for audio content. Each non-comment line has the form:

```
<relative\path\to\file.wav>  [sound_name]  [options...]
```

### Options

| Option | Meaning |
|---|---|
| `-resample <platform> <hz> [<platform> <hz> ...]` | Resample to given rate per platform before encoding |
| `-alias <platform> <target_name>` | On this platform, emit a skip/alias entry pointing to `target_name` instead of including audio |

### Platform-conditional blocks

```
#ifplatform xbox pc
    some_file.wav  sound_name  -resample xbox 22050 pc 44100
#endifplatform xbox pc
#ifplatform ps2
    ps2_file.wav  sound_name  -resample ps2 12000
#endifplatform ps2
```

Entries inside `#ifplatform` are only emitted for the listed platforms. This is the mechanism for
PS2-specific lower-quality WAVs and for choosing which entries carry audio vs. alias on each platform.

### Sound name

- Explicitly given as the second token (before any `-` flags).
- Can be omitted: SoundFLMunge derives the name from the WAV filename (stem, lowercased).
- The FNV-1a hash of the name is stored in the info chunk.

### Alias entries

```
ex2_ord_grenade01.wav   exp_ord_grenade01   -alias ps2 exp_ord_grenade02
```

On PS2 (and by extension PSP), `exp_ord_grenade01` is emitted as a **skip entry** whose
`check2` field (info chunk at SearchStart + 0x1c) stores `fnvHash("exp_ord_grenade02")`.
The game resolves aliases at runtime by scanning the bank for the target hash — no stored offset.

### Compiled output: the .bnk binary format

`SoundFLMunge.exe` writes a UCF file with two chunks per bank:

```
0x0fb40705  info chunk   — tagged (tag, value) uint32 LE pairs
0xd872e2a5  data chunk   — raw audio bytes, entries laid out sequentially
```

Info chunk layout (from disassembly + field analysis):

**Bank header** (first SearchStart block, 5+3 tagged pairs):
- `0x8d39bde6` → dynamic/version value
- `0xb99d8552` → always 4 (format version)
- `0x7816084b` → channel count
- `0x182fd58d` → always 4
- `0x40fbdebd` → wav/sample count in bank
- `0x23a0d95c` → **total audio data size** (SearchStart tag reused)
- `0x7aaf1a1c` → substream count (2 for 4-ch ambient)
- `0x740fdb0c` → substream interleave size (e.g. 0x9000 for Xbox streams)

**Per-entry record** (6 tagged pairs each, relative to SearchStart at position `i`):
- `i - 16`: tag `0x37386ae0` 
- `i - 12`: **name hash** (FNV-1a of sound name)
- `i - 8`:  tag `0x2fb31c01`
- `i - 4`:  **sample rate** (exact Hz)
- `i + 0`:  tag `0x23a0d95c` (SearchStart — reused as data size tag)
- `i + 4`:  **audio data size** (bytes)
- `i + 8`:  tag `0x1d48feef` (stream format flags / unknown)
- `i + 0xc`: value
- `i + 0x10`: tag `0x809608b6`
- `i + 0x14`: **block padding** (alignment bytes after audio data)
- `i + 0x18`: tag `0x2e789fb4`
- `i + 0x1c`: **skip/alias field**:
  - `0x7D268157` = this entry is a skip/alias entry (audio lives in another bank)
  - For skip entries, `i + 0x1c` = `fnvHash(alias_target_name)`
  - For normal entries, `i + 0x1c` = some non-magic value (often 0 or a secondary check)

---

## .stm / .st4 — stream manifests

```
streams\amb_nabooCity_PL2.wav   nab_amb_city   -resample ps2 32000
```

- Each line: `<wav_path>  <stream_segment_name>  [options]`
- `.stm` = mono or PS2 stream; `.st4` = 4-channel (Xbox front+back stereo pair, two WAVs per segment)
- The segment name must match a `Segment("name", ...)` reference in a `.snd` `SoundStreamProperties` block.
- Compiled into the `.str` stream file with 2048-byte block alignment.

---

## .snd — sound property config

Defines game-level sound events. Compiled to `.config` by `configmunge.exe`.

**SoundProperties** — a playable sample-based sound event:
```
SoundProperties() {
    Name("canal");
    Group("ambientenv");
    Inherit("ambientemt_static_template");
    #ifplatform xbox pc
        Gain(0.4);
    #endifplatform xbox pc
    Looping(1);
    SampleList() {
        Sample("emt_canal_stream_lp", 1.0);   // references a name from .sfx
    }
}
```

**SoundStreamProperties** — a stream-based ambient sound event:
```
SoundStreamProperties() {
    Name("nab_amb_city");
    Stream("nab2");              // references .stm/.st4 base name
    SegmentList() {
        Segment("nab_amb_city", 1.0);   // references a segment name from .stm
    }
}
```

**SoundStream** — music stream event (in *_music_config.snd):
```
SoundStream() {
    Name("rep_nab_amb_start");
    Bus("ingamemusic");
    Stream("cw_music");
    SegmentList() {
        Segment("RepFel01_Act02_lp", 1.0, 0.0, 0.0);
        ...
    }
}
```

---

## .mus — music state machine

References `SoundStream` events by name and adds timing/fading parameters:
```
Music() {
    Name("rep_nab_amb_start");
    Priority(1.0);
    FadeInTime(0.0);
    FadeOutTime(1.5);
    MinPlaybackTime(20.0);
    MaxPlaybackTime(180.0);
    SoundStream("rep_nab_amb_start");
}
```

---

## .ffx — foley FX group

Maps a terrain/surface category to a list of sound effect names (from .sfx):
```
FoleyFXGroup() {
    Name("water_foley");
    FoleyFX("cis_inf_droid_water");
    FoleyFX("rep_inf_trooper_water");
    ...
}
```

---

## Summary: what lives where in the final .lvl

```
nab2cw.lvl
├── nab2cw sample bank
│   ├── info chunk (0x0fb40705)  — entry metadata (names, sizes, rates, skip flags)
│   └── data chunk (0xd872e2a5) — raw audio (VAG on PS2/PSP, Xbox ADPCM on Xbox, PCM on PC)
├── global_world.config          — shared game-level sound event table
├── nabcw_music_config.config    — music stream event table
├── nabcw_music.config           — music state machine
├── nab2.config                  — ambient emitter sound props
├── nab2cw_foley.config          — foley terrain mappings
└── [other .config files...]
```

The stream audio (ambient bed, emitters) lives separately in `nab.str`.
