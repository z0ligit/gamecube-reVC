# reVC — GameCube

A Nintendo GameCube port of Grand Theft Auto: Vice City, based on the
reverse-engineered engine from [mrxenginner/reVC](https://github.com/mrxenginner/reVC).

The port targets real GameCube hardware constraints: the heap is limited to
the console's 24 MB of MEM1, audio sample storage uses the 16 MB ARAM, and
MEM2 (Wii-only memory) is never used — including in the Wii development
build, which enforces the same limits.

## Status

Work in progress.

- The game boots and plays from an **SD card** or **USB** (without an SD card inserted, check [Known issues](#known-issues)) (Wii homebrew loader, or
  Dolphin).

## Dependencies
The following packages are required(on Ubuntu 26.04 WSL, package names and packages required to be installed may vary depending on your Linux distribution and release):

- devkitPro(can be installed with [this](https://apt.devkitpro.org/install-devkitpro-pacman) shell script, the webpage won't allow you to wget it, you have to manually copy and paste it to your text editor of choice and then chmod +x the file)
- gamecube-dev and/or wii-dev packages (from dkp-pacman)
- ppc-libvorbisidec and ppc-libogg (from dkp-pacman)
- libogg-dev, libvorbis-dev
- ffmpeg
- sox
- gcc, g++
- cmake
- ninja-build

If you have devkitPro installed in a non-standard location, set the `DEVKITPRO` environment variable.

## Building libtheora
An example program from libtheora is required for converting the intro videos.
```bash
wget http://downloads.xiph.org/releases/theora/libtheora-1.2.0.tar.gz && \
tar axvf libtheora-1.2.0.tar.gz && \
cd libtheora-1.2.0 && \
./configure && \
make && \
export THEORA_ENCODER_EXAMPLE=$(pwd)/examples/encoder_example && \
cd ..
```

Do not move or delete the folder named `libtheora-1.2.0` until after you converted the game files in the [Game data](#game-data) step.

## Building

```bash
git clone --recursive https://github.com/z0ligit/gamecube-reVC
cd gamecube-reVC
python3 build.py wii        # Wii dev DOL  -> build/wii/src/reVC.dol (Works, but not tested thoroughly)
python3 build.py            # GameCube DOL -> build/cube/src/reVC.dol (I couldn't get it working)
```

## Game data
This repository does not contain any game files. A legally owned copy of Grand Theft Auto: Vice City is required.

Copy the game installation to `assets/GTAVC` (the [`assets/`](assets/) folder is git-ignored) and run:

```bash
python3 build.py sd         # SD card tree -> assets/sd-tree
```

This builds the ahead-of-time texture converter for your machine, converts
every texture to GX-native formats, repacks `gta3.img` and lays out the
card tree the game reads (`tools/gamecube/build_sd.py` does the asset
work; `--game`, `--out`, `--audio` and `--movies` override the defaults).

## Running

### Dolphin

1. Build the SD card tree (see [Game data](#game-data)).
2. Point Dolphin's Wii SD card at it: `Config → Wii → SD Card Settings`,
   then either set the SD card image to one whose **root** holds the tree's
   contents, or enable folder sync targeting the tree itself (the sync root
   becomes the card root). Either way the card must end up with
   `/models/gta3.img` at the top level.
3. Open `build/wii/src/reVC.dol` in Dolphin.

### Wii (Homebrew Channel)

1. Generate the SD card tree (see [Game data](#game-data)) and copy its
   **contents** — the `anim/`, `audio/`, `data/`, `models/`, `text/` and
   remaining folders `build_sd.py` produced — directly to the **root** of a
   FAT32 SD card. The game reads them from the root: the card must contain
   `/models/gta3.img`, not `/sd-tree/models/gta3.img`.
2. Copy `build/wii/src/reVC.dol` to the card as `apps/reVC/boot.dol`.
3. Launch the game from a Wii file manager(eg. WiiXplorer).

### GameCube

Real-hardware disc boot is not functional yet — see [Known issues](#known-issues).

## Known issues

The game does not boot up if you play on USB but also have an SD card inserted.
- The game only checks if an SD card is present and is formatted as FAT32. The game only "fails over" to USB if a card is not present and not if the SD card does not contain the game files required to start the game. If you want to play on USB, eject your SD card from the console before starting the game. 

The game only starts if the game files are in the root of the SD card/USB drive.
- Will relocate the to-be-mounted folder to /apps/revcgc/, avoiding a possible conflict with the Wii port. 

Conversion scripts are case sensitive
- I hotfixed it for the copy I have in [this](https://github.com/z0ligit/gamecube-reVC/commit/092f034d32dad4702fc102a3005d3319cec39821) commit, the real solution will be making the conversion scripts case insensitive.

Sometimes during RenderFadingAtomic() the game gives the GetAtomicFromDistance() function a distance variable so large that the if statement inside the for loop will never be true. In those cases the function returns a nil, which crashes the game with a DSI exception every time. the crash was most common around Cortez's boat.
- Hotfixed in [this](https://github.com/z0ligit/gamecube-reVC/commit/f7fb7181fac9c4864a663b50cf69fc986d5c783a) commit, the real solution will be figuring how the game calculates the unrealistic dist variable.

Over budget error by build_sd.py
- The game doesn't use the more space efficient .pak file for sound effects, using it should bring the folder size under the budget. This doesn't cause issues on SD cards or on USB but it should be dealt with before building ISOs.

An untested ISO builder exists in the repo, but it only works on MacOS.
- A more platform inspecific solution is yet to be built.

Error submitting packet to decoder: Invalid data found when processing input
- This doesn't seem to cause any issues, but it is something to investigate maybe.

## Architecture

- **Renderer** — a native GX backend for librw
  (`vendor/librw/src/gx`). Textures are converted ahead of time to
  GameCube-native formats (CMPR / RGB5A3) at full original quality; memory
  pressure is handled by streaming and eviction, not by reducing asset
  quality. World geometry is quantised to packed int16 vertex streams,
  static meshes can be replayed as GP display lists, and lighting is
  implemented with TEV stages (prelight plus timecycle ambient, with
  optional env-map, rim-light and lightmap stages).
- **Audio** — streamed music, radio and speech are Ogg Vorbis, decoded with
  Tremor (fixed-point) on a dedicated thread so decoding never interrupts
  the game frame. Mixing uses AESND's 32 hardware voices. Mission speech
  (IMA ADPCM) is cached in ARAM. FMVs are decoded with Theora.
- **Filesystem and streaming** — an ISO9660 driver written for this port
  (`src/skel/gamecube/dvdfs.c`) plus libfat SD support, with sector-aligned
  DMA reads and a streaming layer tuned for the 24 MB memory budget.
- **Frontend** — a GameCube controls page with a 3D controller model, and
  help boxes that display the port's actual button bindings as coloured
  GameCube button badges.

## Credits

- [mrxenginner/reVC](https://github.com/mrxenginner/reVC) — the
  reverse-engineered Vice City engine this port is based on.
- [librw](https://github.com/aap/librw) by aap — the RenderWare
  reimplementation. [This port's fork](https://github.com/origami-ltd/gamecube-librw)
  adds the GameCube GX backend.
- [dca3](https://gitlab.com/skmp/dca3-game) by skmp and contributors — the
  Dreamcast GTA III port. Specific derivations:
  `tools/gamecube/repack_img.py` is modelled on dca3's imgtool;
  `tools/gamecube/txdconv.cpp` follows its ahead-of-time native texture
  conversion; `tools/gamecube/dffcensus.cpp` reproduces its packed
  native-geometry cost analysis; the pre-instanced static DFF format and
  allocation strategies in `vendor/librw/src/gx/gxraster.cpp` follow
  conclusions established by dca3.
- [Polyphase Engine](https://github.com/Polyphase-Labs/Polyphase-Engine) —
  reference for the GX channel and TEV configuration in
  `vendor/librw/src/gx/gx.cpp`, credited inline where used.
- [GameCube controller 3D model](https://sketchfab.com/3d-models/gamecube-controller-21983501bac64993ac09cdc7936ffdf2)
  by Cory Richards, licensed
  [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Converted to
  RenderWare format in `tools/gamecube/assets/`, licence file included.
- [Xiph.Org](https://xiph.org/) — ogg, opus, opusfile, Tremor and theora.
- [devkitPro](https://devkitpro.org/) — devkitPPC, libogc and AESND.

## License

The port's original contributions are licensed under the
[MIT License with Proof-of-Usage Condition (MIT-PoU)](LICENSE.md). Upstream
components keep their original licenses: librw is MIT (aap), the xiph
libraries are BSD, and code inherited from reVC remains under its upstream
terms.
