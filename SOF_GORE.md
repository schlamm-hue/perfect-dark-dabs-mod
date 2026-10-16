# Soldier of Fortune Gore — Mod für Perfect Dark (DABS-Mod)

Blut-Spray und Blutpfützen nach dem Vorbild von **Soldier of Fortune 1 (ProGore)**:
Die Menge des Bluts skaliert mit der Zone, in die der Schuss geht — Kopfschüsse
spritzen, Beintreffer bluten dezent.

Basis: [`DabDavis/perfect-dark-dabs-mod`](https://github.com/DabDavis/perfect-dark-dabs-mod) (MIT, vollständiger C-Quellcode).
Zonen-Tabelle und Spray-Logik stammen aus der REA/Ghidra-Analyse von SoF1s `gamex86.dll`
(Doku: `GORE_SYSTEM.md`, Port-Design: `GORE_PORT_PD.md`).

## Aktivieren

Die Option ist **ab Werk an**. Abschalten geht auf zwei Wegen:

| Weg | Wie |
|---|---|
| Im Spiel | **Options → Player-Seite → „Soldier of Fortune Gore“** (Checkbox, wie „Flinch When Shot“) |
| Datei | `pd.ini` → `Mod.SofGore=0` bzw. `1` |

Im Multiplayer ist die Option als **Room-Regel** sichtbar und wird vom Gastgeber gesetzt.

## Was der Modus ändert

Nur die **Menge/Größe des bestehenden Bluteffekts** — keine neuen Assets, keine Physik-Objekte,
kein Dismemberment. Aus = Originalspiel, unverändert.

| Effekt | Verhalten mit Modus |
|---|---|
| Blut-Spray pro Treffer | `1 + Zonen-Multi` Bursts (Blood + Fleisch), Richtung = Schussvektor, Position = exakte Trefferstelle |
| Grobfleisch-Spritzer (FLESH_LARGE) | Trefferchance skaliert mit Zone (Kopf ~32 %, Fuß ~12 %) |
| Blutpfützen/-tropfen pro Treffer | Menge mit Zonen-Multi des getroffenen Körperteils |

## Zonentabelle (SoF ProGore → Perfect Dark)

PD trackt 16 Trefferpartien (`HITPART_*`), SoF 28 Zonen. Gemappt auf:

| PD-Partie | SoF-Zonen | Multi |
|---|---|---|
| HEAD | GZ_HEAD, HEAD_FRONT/BACK, NECK | **1.6** |
| TORSO | GZ_CHEST_FRONT/BACK, GZ_GUT_FRONT/BACK, GZ_GROIN | **1.4** |
| PELVIS | GZ_GROIN, GZ_LHIP, GZ_RHIP | 1.2 |
| THIGH / SHIN / BICEP / FOREARM | LEG/ARM Zonen | 1.0 |
| FOOT / HAND | GZ_FOOT_*, Hand | 0.6 |
| GUN / HAT (Rüstung) | gz_armor, gz_headhelmet | — (kein Extra-Spray) |

Tabelle zum Nachjustieren: `g_SofGoreZoneMul[]` in `src/game/modoptions.c` — reine Daten.
Feiner (Partikelzahl/Tempo/Farben pro Spray): `g_SparkTypes[]` in `src/game/sparks.c`
(Standard: Blood 40 Partikel dunkelrot, Fleisch 40 mit Speed 300/200, Grobfleisch 10 mit Speed 1200).

## Build (Linux/WSL)

```bash
sudo apt-get install -y libsdl2-dev zlib1g-dev libcurl4-openssl-dev
git clone https://github.com/DabDavis/perfect-dark-dabs-mod.git pd-dabs
cd pd-dabs && git submodule update --init --depth 1
# ROM nach data/pd.ntsc-final.z64 legen
cmake -G"Unix Makefiles" -Bbuild . && cmake --build build -j6
./build/pd.x86_64
```

Windows: dem offiziellen Release/Build-Weg des DABS-Mods folgen (MSYS2/MSVC).

### WSLg-Maus-Inversion (bekannt)

Der WSLg-Compositor liefert SDL2-Raw-Input-Deltas invertiert. Fix beim Start:

```bash
SDL_MOUSE_RELATIVE_MODE_WARP=1 ./build/pd.x86_64
```

## Geänderte Dateien (Commits `6248714`, `4ee539d`)

| Datei | Inhalt |
|---|---|
| `src/include/game/modoptions.h` | Option `sofgore` + Getter-Deklarationen |
| `src/game/modoptions.c` | Default (an), Getter, Zonentabelle `g_SofGoreZoneMul` |
| `src/game/chr.c` | `chrEmitSparks`: zonen-skalierter Spray-Zweig |
| `src/game/splat.c` | `splatsCreateForChrHit`: zonen-skalierte Pfützenmenge |
| `port/src/main.c` | ini-Key `Mod.SofGore` |
| `port/src/optionsmenu.c` | Menü-Checkbox + Room-Regel |

## Offen / P3-Kandidaten

- GUT-Zone separat behandeln (PD hat nur TORSO, SoF trennt Chest/Gut)
- Multi-Hit-Skalierung (SoFs `gz_multi%d`: Spray wächst bei Trefferserien)
- Blood-Textures/Sounds aus SoF-Assets (N64-Formatkonverter nötig)
- Gibs/Dismemberment: eigenes Projekt (PD hat kein Gib-System, N64-Prop-Physik nötig)

## P3 + P4 (Stand 2026-10-16, Commits `a876d6e`, `923fb6d`)

**P3 (fertig):** Multi-Hit-Ramp `modSofGoreHitRamp` (pro `chrnum`, Fenster über
`g_Vars.updateframe` — Feld heißt NICHT `updatetime`), Kill-Pool + beschleunigtes
Leichen-Bluten in `splat.c` (`ACT_DIE`/`ACT_DEAD`, kein `chr->health`-Feld).

**P4 (fertig): Gib-Billboards.** PD hat kein Gib-System; Dismemberment ist nicht
machbar (fixe Skin-Meshes). Lösung über das Smoke-System:
- `SMOKETYPE_GIB` (23) / `SMOKETYPE_GIB2` (24) in `constants.h`, Tabellenzeilen in
  `g_SmokeTypes[]` (PAL **und** NTSC-Zweig; `unk1c` = Vertikalgeschwindigkeit negativ
  = fällt, `unk18` = Wachstum, `unk20` = Drift).
- `g_TcGibConfig` in `textureconfig.c`: Texturnummer **0x0c97** =
  `g_TcGeneralConfigs[2]` — kein Call-Site nutzt Index 2, Nummer ist exklusiv.
- `smokeRender`: für Gib-Typen `texSelect(&gdl, &g_TcGibConfig, 4, 0, 2, 1, NULL)`
  statt des geteilten `g_TexGdl1` (alle Smokes teilen sonst EINE Textur).
- Spawn: `chrEmitSparks` (mul ≥ 2.0 → Head/Gut, 1–2 Chunks entlang Schussrichtung)
  und Kill-Burst (4 Chunks bei `ACT_DIE`) in `splat.c`.
- Art: `texpack-staging/sof-gore/0c97.png` ← SoF `chunks_meat/chunk1.tga`
  (16×16 Graustufen+Alpha, Luminanz normalisiert; Rauchfarbe tintet rot).
- Texpack-Setup: `build/texture-packs/sof-gore/` + `pd.ini` `TexturePack=sof-gore`
  (`Mod.LoadTextures=1`); F8 toggle / F9 reload / F10 cycle.
- Decals (Ebene 1): `0854/0855/0856` ← bldsprinkle/bldglob4/bldglob5,
  `08f0` ← blood_pool, `001a` Spark-Strich. Konverter: `sof-gore-port/m32_to_texpack.py`
  (m32 = 516 B Header + Mip-Breitenliste + Offsettabelle, Pixdaten roh RGBA).

**Nicht machbar:** echtes Dismemberment (Modellvarianten pro Glied nötig),
3D-Gibs mit Physik (neue F3DEX-Prop-Modelle = eigenes Projekt).
