# DreamMix TV World Fighters – scfight.zb extractor

Python extractor for the `scfight.zb` archive from **DreamMix TV World Fighters** (GameCube, Japan, `GKWJ18`, Hudson / Bitstep, 2003).

`scfight.zb` holds almost all of the game's data: models, textures, animations, effects, parameters, messages and the MusyX sound bank. The script decrypts the file table and writes every file out under its original name and folder layout.

## Requirements

- Python 3.6+
- No third-party modules
- **main.dol is not needed**: the decryption algorithm and its keys have been reverse-engineered and are built into the script.

## Getting scfight.zb

Get the game and `scfight.zb` through your own way.

## Usage

```sh
python zb_extract.py scfight.zb                # extracts to ./scfight/
python zb_extract.py scfight.zb -o my_folder   # extracts to ./my_folder/
python zb_extract.py scfight.zb --list         # lists the files without extracting
python zb_extract.py scfight.zb --keep-prefix  # keeps the full /home/project/ScFight/GCN/... path
```

By default the development prefix `/home/project/ScFight/GCN/` is stripped, which gives you:

Expected result with the Japanese version: **1902 files**, 1663 of which are zlib-compressed, about 110 MB.

## Archive format

All integers are **big-endian**.

### Header (0x20 bytes, plaintext)

| Offset | Type | Description |
|---|---|---|
| 0x00 | u32 | magic `0x0131A36E` (= 20030318) |
| 0x04 | u32 | version `0x32` |
| 0x08 | u32 | number of files |
| 0x0C | u32 | TOC size (the TOC runs from 0x20 to 0x20 + size) |
| 0x10 | u32 | key seed: high 16 bits = A, low 16 bits = B |
| 0x14 | u32 | key seed: K |
| 0x18 | u32 | timestamp (time_t) |
| 0x1C | u32 | timestamp (duplicate) |

### TOC entry (encrypted)

| Type | Description |
|---|---|
| cstring | path; each character is XORed with the low byte of the keystream (unless it already equals that byte); the terminating `\0` is not encrypted |
| u32 ^ K | flags: low byte `'z'` = zlib, `'n'` = stored |
| u32 ^ K | uncompressed size |
| u32 ^ K | stored size |
| u32 ^ K | absolute offset of the data |
| u32 | unknown (plaintext, always 0) |
| u32 | unknown (plaintext, always 0) |
| cstring | second string, encrypted like the path (always empty) |

Every encrypted byte or word advances the keystream by **one step**.

### Keystream

```
A = seed >> 16 ; B = seed & 0xFFFF ; K = key_k
step:
    bit = ((A & 0x0008) == 0x0008) XOR ((A & 0x4000) == 0x4000)
    A   = (2*A + !bit) & 0x7FFF
    bit = ((B & 0x0004) == 0x0004) XOR ((B & 0x4000) == 0x4000)
    B   = (2*B + !bit) & 0x7FFF
    A   = 5*A + 1 ; B = 5*B + 1
    K  ^= A | (B << 15)
    return K
```

The file data itself is not encrypted: it is either stored as-is or stored as a zlib stream (the game uses zlib 1.1.3).

### Where this comes from in main.dol (GKWJ18)

- `0x80110918`: TOC reader (`hi_fileio.c`)
- `0x80208688`: table of LFSR constants
- `0x801119cc`: archive initialisation (`/scfight.zb`)

## Formats of the extracted files (for reference)

| Extension | Format |
|---|---|
| `.cs` | RenderWare 3.x: custom texture dictionary (chunk `0x23`, containing standard RwImage `0x18` chunks) followed by a standard Clump (`.dff`) |
| `.txd` | same `0x23` texture dictionary |
| `.as` | RenderWare animation (chunk `0x1B`) |
| `.db`, `.efo`, `.eff` | text |
| `.msg` | message table (`MSG\0`) |
| `.pool .proj .samp .sdir .song` | Nintendo MusyX sound bank |
