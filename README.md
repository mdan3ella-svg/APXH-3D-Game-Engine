# APXH-3D-Game-Engine
HTML 5 WebGl 3D game Engine
# GAME ΑΡΧΗ ENGINE™
### VERTASCAN // Where Every Game Begins

**Current build: v0.4 TETRA** · single-file · zero-install · 2D + 3D · ~262 KB

ΑΡΧΗ (*arche* — Greek: "origin, first principle") is a complete game creation environment that lives in **one HTML file**. Open it in a browser and you have a 2D engine, a 3D engine, a visual editor for both, an asset pipeline, and a one-click export system that produces standalone, publishable HTML5 games. No installs, no build step, no node_modules — the engine *is* the file.

---

## Version Lineage

| Version | Codename | Layer added |
|---------|----------|-------------|
| v0.1 | — | Canvas2D core: scene graph, physics, editor |
| v0.2 | **PRISM** | WebGL2 renderer, native GLB parser, backgrounds, ambient environments, full audio system |
| v0.3 | **TRIAX** | Three.js 3D Sandbox: prop building, animated GLB, player physics, video screens, 3D export |
| v0.4 | **TETRA** | 3D gameplay layer: enemy AI, combat, pickups, keys/doors, triggers, checkpoints, win/lose, HUD |

Codenames follow the Greek numeral stem of the version (TRI → 3, TETRA → 4).

---

## Architecture

```
index.html
├── <style>              UI shell (Vertascan design system)
├── <script> 2D CORE     editor + runtime, Canvas2D + WebGL2, zero dependencies
├── <style> + markup     3D Sandbox overlay (#app3d)
├── <script id="arxe3d-runtime">   ARXE3D module — embedded verbatim in 3D exports
├── <script> 3D EDITOR   sandbox editor, serialize/load bridge, export builder
└── <script> tail sentinel (boot-integrity check)
```

**Dependency policy:** the 2D engine has *zero* external dependencies and runs fully offline. The 3D layer lazy-loads **Three.js r128 + GLTFLoader** from CDN only when the sandbox is opened (or when an exported 3D game boots). If the CDN is unreachable the 2D side is unaffected and the 3D overlay reports `3D CORE OFFLINE` instead of breaking.

**Boot integrity:** a tail sentinel (`window.__arxeTail`) verifies the file survived transfer intact — the engine detects truncation/corruption (e.g. from pasting into a web editor) and reports it instead of silently hanging.

---

## Part I — 2D Engine (ΑΡΧΗ CORE)

### Editor
Panel layout: scene graph (left) · live viewport (center) · tabbed inspector (right) · console/log strip. Undo stack, node duplication, keyboard shortcuts, project name field, save/load/export in the top bar.

### 2D Node Types

| Type | Purpose |
|------|---------|
| `rect` | Solid/platform primitive |
| `circle` | Round primitive |
| `sprite` | Procedural vector sprite — patterns: `hull`, `grid`, `star`, `orb`, `ship`, `core` |
| `text` | Styled text element |
| `particles` | Emitter node (presets incl. sparks, snow, rain, embers, ash, dust, void, plasma, leaves) |
| `camera` | View window (960×540 default) |
| `mesh` | **WebGL2 3D mesh in the 2D scene** — built-in models or imported GLB, with `zdepth`, `spin`, `pitch`, `meshScale`, `yawFix` |

Every node carries transform (`x y rot z`), size, color, visibility, a free-text `tag` for logic targeting, a physics block, a logic list, and an optional script.

### Physics (per node)
AABB physics with `body: dynamic | static`, per-node `gravity` scale and `bounce`, initial `vx/vy`, and a `trigger` flag (overlap events without collision response). Scene-level gravity (default 1400 px/s²).

### Logic System — no-code events & actions
Each node holds a list of `{event, action}` rules built from dropdowns:

**Events:** `On start` · `Every frame` · `Key held…` · `Key pressed…` · `Every N sec…` · `On collide…` (filter by node/tag) · `On click`

**Actions:** `Move X/Y (px/s)` · `Set vel X/Y` · `Jump (impulse)` · `Chase node/tag…` · `Spawn clone of…` · `Spawn at edge…` · `Destroy self` · `Destroy other` · `Play SFX…` · `Add score` · `Particle burst…` · `Show message…` · `Play soundtrack…` · `Stop soundtrack` · `Restart scene`

That vocabulary covers platformers, arena shooters, chase games, spawners, and score loops without writing code.

### Script hooks — full-code escape hatch
Every node also has a JS `script` field that may assign `self.onUpdate(dt)` and `self.onCollide(other)` for behavior beyond the rule system.

### Scene environment
- **Backgrounds:** `solid | gradient | stars | nebula | grid | image` with `fit` (cover/contain/stretch/tile), `parallax` (0 = screen-locked → 1 = world-locked) and a `dim` veil for readability.
- **Ambient environments (`env`):** one-click atmospheric particle presets — `snowfall`, `rainfall`, `embers`, `ashfall`, `dustmotes`, `voidmist`, `plasma`, `leaffall` — with adjustable rate.
- **Audio:** looping scene soundtrack + WebAudio SFX bank, volume control.

### 2D Export
**Export HTML5** produces a single standalone file with the runtime embedded verbatim and every asset (images, audio, GLB) baked in as base64. 2D exports are **fully offline-capable**.

---

## Part II — 3D Sandbox (ARXE3D)

Opened from the top bar (**3D Sandbox**). A separate full-screen editor backed by the `ARXE3D` module — the same module that ships inside every 3D export, so play-in-editor behavior is *identical* to the published game.

### 3D Node Types

| Type | Purpose | Default solid |
|------|---------|:---:|
| `box` `sphere` `cylinder` `cone` `pad` | Primitive props (scale, color) | ✔ |
| `ramp` | True wedge geometry — walkable slope with yaw-aware height | ✔ |
| `lamp` | Point light (color, intensity, range) | ✘ |
| `glb` | Imported GLB model, uniform scale, animation clip + speed | ✔ (bounding box) |
| `video` | In-world screen playing an imported .mp4/.webm (loop/audio/autoplay flags) | ✔ |
| `spawn` | Player start marker (editor-only cone) | ✘ |
| `enemy` `pickup` `trigger` `goal` `checkpoint` `door` | **Gameplay layer — see Part III** | door only |

### Environment
Gradient sky (canvas texture) · distance fog · hemisphere ambient + shadow-casting directional sun (color, intensity, angle) · optional ground plane (color, size) · looping music + ambient audio loop with independent volumes.

### Player Physics (capsule vs. AABB)
- Capsule collider: `radius 0.38`, `height 1.7` (configurable)
- Horizontal wall resolve via closest-point push-out; degenerate-center fallback along the shortest axis
- **Step-up:** ledges ≤ 0.5 u are climbed automatically
- **Ramps:** yaw-aware wedge height function — walk up rotated ramps correctly
- Head-bump clamp when jumping into overhangs
- Fall below **y = −60** → respawn (v0.4: at last touched checkpoint, else spawn)
- Camera-relative WASD movement; tunable `speed`, `jump` impulse, `gravity`

### Cameras & Character
`third` (orbit-follow, distance/height tunable) or `fps` (model hidden). Player is either the default capsule pilot or **any imported GLB** with clip slots for idle / run / jump / attack, crossfaded automatically. **Forward-axis fix (Mixamo)** toggle applies the house yaw convention (`rotation.y = π`) to player and enemy rigs.

### Asset Pipeline
`+ GLB` / `+ Audio` / `+ Video` import via FileReader → base64 data URLs stored in per-kind asset banks. GLBs are pre-parsed on import so **animation clip names auto-populate every dropdown**. Assets embed into saves and exports — sub-5 MB GLBs recommended to keep single-file builds lean.

### Editor Viewport Controls
| Input | Action |
|-------|--------|
| LMB | Select / drag on ground plane (0.5 u snap) |
| Alt + drag | Raise/lower (0.25 u snap) |
| RMB drag | Orbit |
| Shift + LMB / MMB | Pan |
| Wheel | Zoom |

---

## Part III — Gameplay Layer (v0.4 TETRA)

### Enemy AI
Per-enemy state machine: **idle → patrol → chase → attack → dead**
- **Patrol:** random wander inside `patrol` radius around spawn point, with idle pauses
- **Chase:** pursues when player enters `detect` range (full `speed2`; patrol moves at half)
- **Attack:** in `atkRange`, faces the player and deals `dmg` every `atkRate` seconds
- Gravity + ground/box-top floor tracking, wall push-out, fall-out auto-reset
- Per-enemy `hp`; on death plays the death clip or capsule-topple fallback, then despawns
- Visuals: red capsule sentinel or any GLB with idle/run/attack/death clip slots (yaw-fix applies)

### Player Combat
Melee arc attack — forward-facing dot-product check within `atkRange`:
- **Inputs:** `F` / `E`, mouse click while pointer-locked, or the **ATK** touch button
- Configurable damage / range / cooldown (GAME tab) + attack animation clip (PLAYER tab)
- **Sprint:** `Shift` × 1.55 speed
- Taking damage: 0.6 s i-frames, knockback, red vignette flash; HP 0 → lose

### Interactive Nodes
| Node | Behavior |
|------|----------|
| `pickup` | `coin` (score, counts toward collect-win) · `heart` (heals 25 × value) · `key` (door currency). Spinning/bobbing built-in shapes or custom GLB |
| `trigger` | Invisible volume (editor shows tinted wireframe). Actions: **message · damage · heal · teleport · win · lose**, with amount/target fields. `FIRE ONCE` or repeating (1 s refire guard) |
| `door` | Solid sliding panel; unlocks (and stops colliding) when the player approaches with ≥ `needKeys`, consuming them. Locked prompt otherwise |
| `checkpoint` | Flag pole; touching sets the fall-respawn point |
| `goal` | Emissive pad + spinning ring; wins the game in `goal` mode |

### Win / Lose (GAME tab)
- **Win conditions:** `Reach the GOAL pad` · `Collect every coin/orb` · `Defeat every enemy` · `Sandbox (none)`
- Mission title + objective text (shown at start), custom win/lose messages
- Player max HP, attack tuning, and **five gameplay SFX slots**: pickup, hurt, attack swing, win sting, lose sting

### HUD (auto-built, DOM overlay)
Mission title · HP bar (turns red under 35%) · context stats line (SCORE / ORBS x/y / KEYS / TARGETS x/y per win mode) · center notification toasts · damage vignette · end overlay with stats and **RESTART** (button or `R`). Toggleable per project. Identical in editor play-test and exports.

---

## Data Formats

**Project save — `ARXE_PROJECT` v2** (2D Save button): `{ format, version: 2, name, scene, three }` — the `three` key carries the entire 3D world so one `.json` holds both editors' state.

**3D world — `ARXE3D_WORLD` v2**: `{ world, player, game, nodes[], assets: { glb, audio, video } }`.

**Forward migration:** `guard()` deep-merges defaults into any loaded data and **backfills all v0.4 gameplay fields onto v0.3-era saves and nodes** — old projects load clean, no manual upgrade.

---

## Export Pipeline

| | 2D — Export HTML5 | 3D — Export 3D Game |
|---|---|---|
| Output | one `.html` file | one `.html` file |
| Runtime | embedded verbatim | `#arxe3d-runtime` embedded verbatim + world JSON |
| Assets | base64-embedded | base64-embedded |
| Offline | **fully offline** | needs connection at boot (Three.js CDN) |
| Controls | keyboard | WASD + mouse-lock, `F`/click attack, `Shift` sprint, `Space` jump; touch joystick + JUMP + ATK |

3D export hardening: split `'<scr'+'ipt>'` literals and `</script` → `<\/script` JSON escaping so embedded data can never terminate the document early. Exports open on a branded start screen (title, controls hint, click-to-start — which also satisfies mobile audio/pointer-lock gesture requirements).

---

## Deployment (house rules)

1. Deploy to **GitHub Pages** via **Add file → Upload files (drag-and-drop)**. Never paste file contents into the GitHub web editor — large single-file builds get corrupted, producing boot hangs. The tail sentinel exists to catch exactly this.
2. Keep GLBs under ~5 MB each; everything embeds as base64 (~×1.37 size).
3. One file = one game. Saves (`.json`) are the working format; exports are the shipping format.

---

## Branding & Design System

**Marks:** `GAME ΑΡΧΗ ENGINE™` · `VERTASCAN` · tagline **"Vertascan // Where Every Game Begins"** · sandbox mark `ΑΡΧΗ 3D — SANDBOX // TETRA`. Exports credit **"built with Game ΑΡΧΗ Engine"**.

**Typography:** **Orbitron** (700/900) for display · **Space Mono** for UI/code/HUD.

**Palette:**

| Token | Hex | Role |
|-------|-----|------|
| Void | `#04060b` | Background |
| Panel | `#0d1220` | Surfaces |
| Text | `#c8d3e8` | Primary text |
| Dim | `#5d6a85` | Secondary text |
| Cyan | `#4fd1c5` | Primary accent, player, success |
| Orange | `#ff7a1a` | Action accent, jump |
| Violet | `#b26bff` | Assets, triggers |
| Red | `#ff2d3a` | Enemies, danger, lose |
| Gold | `#ffd27a` | Pickups, keys, score |

**Motifs:** clipped-corner (chamfer) buttons, letter-spaced uppercase micro-labels, wireframe editor markers, scanline-era console log strip.

---

*GAME ΑΡΧΗ ENGINE™ v0.4 TETRA — © Vertascan. One file. Every game.*
