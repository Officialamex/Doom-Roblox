# Doom for Roblox: a Luau port of the DOOM engine

A function-by-function port of id Software's GPL release of the DOOM source code
(`linuxdoom-1.10`) to Luau, running inside Roblox. The game logic and the original
software renderer run on the client in Luau. Each frame is drawn into a 320×200 8-bit
framebuffer and shown through an `EditableImage`, so what you see comes from the
original column-and-span renderer, not from Roblox parts and meshes.

All game data comes from [Freedoom](https://freedoom.github.io), a free, BSD-licensed
replacement for the original game's levels, art, sound effects and music.

<img width="1920" height="1080" alt="Screenshot 2026-10-11 at 1 02 13 AM" src="https://github.com/user-attachments/assets/fc1e2266-1b24-459c-8e12-2b01969dc2d4" />


## What works

- **Renderer:** BSP traversal, textured walls, perspective floors and ceilings, sky,
  sprites with 8 rotations, masked mid-textures, sector lighting with distance
  falloff, the spectre fuzz effect, and the weapon overlay.
- **Game logic:**
  - Player movement, collision and sliding.
  - Doors, lifts, crushers, stairs, lights, switches and teleporters.
  - Pickups.
  - Monster AI with sight checks and sound alerts.
  - Hitscan and projectile weapons.
  - Death and level restart.
- **Status bar and HUD:** the status bar with ammo, health, armor, weapons, keys and
  the animated face, plus pickup messages and the damage/bonus palette flashes.
- **Sound:**
  - A port of `s_sound.c`: 8 channels, priorities, one sound per source, distance
    attenuation.
  - Playback through Roblox `Sound` objects, plus looping level music.
- **Map:** E1M1 is fully playable.

## Not included (yet)

- Levels other than E1M1. Finishing the level shows a short tally, then restarts.
- Menus, automap, save/load, cheats, demos and multiplayer.
- Stereo panning: Roblox 2D sounds have no pan, so volume falls off with distance but
  there's no left/right separation.

## How it works

| Piece | What it does |
|---|---|
| `info.c` | id's original state and thing tables are stored verbatim and parsed at startup. All 967 states, 137 thing types and their action functions come straight from that file. |
| WAD data | A one-time importer downloads `freedoom1.wad` from the Freedoom release inside Studio, keeps the ~1,100 lumps E1M1 needs (3.09 MB), checks a checksum, and stores them as base64 ModuleScripts. |
| Fixed-point math | Doom's 16.16 fixed point and 32-bit angles run on Luau doubles: shifts become floor division, and angles wrap modulo 2³². |
| Display | 8-bit framebuffer → palette lookup → RGBA buffer → `EditableImage:WritePixelsBuffer`. The image is shown at 4:3 with nearest-neighbour scaling, like the original on a CRT. |
| Timing | Game logic runs at Doom's fixed 35 tics per second. Frames are rendered on `RenderStepped`. |
| Audio | All sound effects are packed into one uploaded audio file (`audio/DoomSfx.ogg`). Each sound is played as a slice of it using `Sound.PlaybackRegion`. Offsets are in `audio/SfxTable.luau`. |

## Repository layout

```
src/        Engine modules (one ModuleScript each)
studio/     DoomClient.client.luau (LocalScript), Inflate.luau and ImportFreedoom.luau (data importer)
audio/      DoomSfx.ogg, DoomMusicE1M1.ogg, SfxTable.luau
tools/      select.py (choose lumps), sfxsheet.py (build the sound sheet), sf2render.py (render MIDI music)
test/       Headless tests for Lune
licenses/   GPL v2, Freedoom BSD license, GeneralUser GS license
docs/       Screenshot
```

### Engine modules and the C files they port

| Module | Original source |
|---|---|
| `Core` | m_fixed.c, tables.c, m_random.c, m_bbox.c, doomdef.h constants |
| `Info` | info.c (parsed at runtime), d_items.c |
| `Data` | w_wad.c, r_data.c, sprite setup from r_things.c |
| `Setup` | p_setup.c |
| `MapUtil`, `Map`, `Sight` | p_maputl.c, p_map.c, p_sight.c |
| `Mobj`, `Tick`, `Inter`, `User` | p_mobj.c, p_tick.c, p_inter.c, p_user.c |
| `Enemy`, `Pspr` | p_enemy.c, p_pspr.c |
| `Spec`, `Doors` | p_spec.c, p_lights.c, p_switch.c, p_telept.c, p_doors.c, p_plats.c, p_floor.c, p_ceilng.c |
| `Render` | r_main.c, r_bsp.c, r_segs.c, r_plane.c, r_things.c, r_draw.c, r_sky.c |
| `Video`, `Status`, `HUD` | v_video.c, st_stuff.c, hu_stuff.c |
| `Game` | g_game.c, d_main.c (single-player subset) |
| `Sound` | s_sound.c, sounds.c |

All modules share one namespace table `D`. Every C global becomes `D.<SameName>`,
so the Luau code can be read side by side with the original C.

## Setting it up in Roblox Studio

1. **Engine:** for each file in `src/`, create a ModuleScript named after the file
   (without `.luau`) under `ReplicatedStorage.Doom.Engine`.
2. **Game data:** create `ReplicatedStorage.DoomImport.Inflate` from
   `studio/Inflate.luau`. Then paste `studio/ImportFreedoom.luau` into the command bar
   and run it. It downloads the Freedoom data, verifies it, writes
   `ReplicatedStorage.Doom.Data`, and switches HTTP back off. Progress is in
   `_G.DoomImport.stage`.
3. **Client:** add `studio/DoomClient.client.luau` as a LocalScript named `DoomClient`
   in `StarterPlayer.StarterPlayerScripts`.
4. **Settings:**
   - Set `Players.CharacterAutoLoads` to `false`. The game draws its own player view.
   - Turn on **Game Settings → Security → Allow Mesh / Image APIs**. Without it the
     client can't draw and shows these instructions on screen instead.
5. **Audio:**
   - Upload `audio/DoomSfx.ogg` and `audio/DoomMusicE1M1.ogg` (Asset Manager → Bulk
     Import).
   - Create a Folder `ReplicatedStorage.Doom.Audio`, put `audio/SfxTable.luau` in it as
     a ModuleScript named `SfxTable`, and give the folder two number attributes:
     `SfxSheet` (the sound sheet's asset ID) and `MusicE1M1` (the music's asset ID).
   - Missing IDs just mean silence.
6. Press **Play**.

> Roblox limits `EditableImage` in published experiences. Check Roblox's current
> requirements before publishing.

## Controls

| Action | Keys |
|---|---|
| Move | W/S or ↑/↓ |
| Strafe | A/D |
| Turn | Mouse, or ←/→ |
| Run | Shift |
| Fire | Left mouse button or Ctrl |
| Use / open | Space, E or right mouse button |
| Weapons | 1–7 |

## Testing outside Roblox

The engine modules use no Roblox APIs, so they run headlessly under
[Lune](https://github.com/lune-org/lune). The tests read `freedoom1.wad` and id's
`info.c` from hardcoded paths. Search `test/*.luau` for `/root/` and change those paths
to match your machine.

```
lune run test/load.luau      # loads every module, reports undefined names and missing actions
lune run test/render.luau    # boots E1M1 and writes one frame as raw RGBA
lune run test/play.luau 2100 # scripted run (number of tics): movement, combat, pickups
lune run test/combat.luau    # pistol cycle and zombieman AI
lune run test/sound.luau     # sound logic with stub audio hooks
```

## Deviations from the original

- Fixed-point products are computed in doubles. Results match 32-bit C except in
  extreme overflow cases, so demo playback compatibility isn't a goal.
- Wall textures wrap by their real height instead of `& 127`.
- Visplane, drawseg and sprite limits aren't enforced, so the vanilla overflow
  crashes can't happen.
- Random sound pitch variation is off by default, as in DOS v1.9 (`D.snd_pitchperturb`).
- Vanilla gameplay quirks are kept, such as monsters refusing to make an immediate
  180° turn.

## Credits and licenses

- **Engine:** derived from id Software's DOOM source release
  (<https://github.com/id-Software/DOOM>), licensed under the
  **GNU General Public License v2**. This port is distributed under the same license
  (`licenses/GPL-2.0.txt`). If you redistribute it, keep the notices and make the
  source available.
- **Game data, sound effects and music:** © 2001-2024 Contributors to the Freedoom
  project, **BSD license** (`licenses/FREEDOOM-BSD.txt`).
- **Music rendering:** done with the **GeneralUser GS** SoundFont by S. Christian Collins
  (<https://www.schristiancollins.com/generaluser>), see `licenses/GENERALUSER-GS.txt`.

DOOM is a trademark of id Software. This project is not affiliated with or endorsed by
id Software, ZeniMax or Roblox, and contains none of the original commercial game data.
