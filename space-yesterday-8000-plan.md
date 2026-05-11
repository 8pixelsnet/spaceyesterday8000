# Space Yesterday 8000 — Implementation Plan

A browser HTML5 space shooter rendered **entirely with code** (no external images, no internet assets). Pixel-art / voxel aesthetic on a 2D canvas, retro cosmic galaxy vibes.

---

## 1. Tech Stack & Constraints

- **Single self-contained file**: `index.html` with embedded `<style>` and `<script>`. No build step, no dependencies, no external assets, no fonts loaded from the web. Drop into any browser, double-click, it runs.
- **Rendering**: HTML5 Canvas 2D API. One main canvas, fixed logical resolution (e.g. `480x270` scaled up via CSS `image-rendering: pixelated` for crisp pixel-art look on any screen size).
- **No images, no SVG files, no audio files**. Every visual is drawn from primitives (rects, paths, gradients) or computed pixel-by-pixel from voxel arrays.
- **Game loop**: `requestAnimationFrame` with fixed-timestep update (e.g. 60 ticks/sec) and interpolated render.

> **Open question for you to confirm:** Audio was not mentioned. Plan assumes **silent game**. If you want chiptune-style sound, say so — it can be done with the WebAudio API (no files), but it adds work.

---

## 2. File Structure

```
index.html        ← everything lives here
```

Inside the `<script>` tag, organize as logical modules separated by comment banners:

```
// ===== CONFIG =====
// ===== INPUT =====
// ===== RENDER HELPERS (voxel, pixel-grid, glow, parallax) =====
// ===== TITLE (3D voxel "Space Yesterday 8000") =====
// ===== ENTITIES (player, bullets, enemies, powerups, particles) =====
// ===== ENEMY DEFINITIONS (dog, piano, etc.) =====
// ===== BOSS (cosmic clock) =====
// ===== POWERUPS =====
// ===== EXPLOSION SYSTEM =====
// ===== STATE MACHINE (menu, play, boss, credits) =====
// ===== GAME LOOP =====
```

---

## 3. Visual Rendering Approach — How to Draw Everything With Code

This is the hardest part of the spec. Three rendering primitives cover everything:

### 3a. Pixel-grid sprites
Define sprites as 2D arrays of palette indices. Render by looping and filling small squares (e.g. 3x3 logical pixels each).

```js
// Example: a 5x5 alien
const alien = [
  [0,1,1,1,0],
  [1,1,2,1,1],
  [1,2,2,2,1],
  [1,1,2,1,1],
  [0,1,0,1,0],
];
function drawPixelSprite(ctx, sprite, x, y, palette, scale) { ... }
```

Use this for: enemies (dog, piano), powerup icons, player ship, bullets.

### 3b. Voxel sprites (for the title and optional 3D enemies)
Define as 3D arrays `[z][y][x]` of palette indices. Render by iterating from back voxel to front voxel, drawing each as an **isometric cube** (a top rhombus + left-face quad + right-face quad with three shades of the same color for fake lighting).

```js
function drawVoxel(ctx, x, y, z, color, voxelSize) {
  // draws 3 quads forming an iso cube at projected (x,y,z)
}
function drawVoxelModel(ctx, model, screenX, screenY, voxelSize) {
  // sort back-to-front, call drawVoxel per filled cell
}
```

Iso projection: `screenX = (x - z) * voxelSize`, `screenY = (x + z) * voxelSize/2 - y * voxelSize`.

### 3c. Procedural / vector primitives
Used for: galaxy background, the cosmic clock boss, explosions, glow halos, the title's neon trail.

- **Galaxy background**: 3 parallax star layers (random `Math.seedrandom`-style fixed seed so stars don't reshuffle every frame), plus 2–3 distant nebula blobs drawn with radial gradients (purple, magenta, deep blue), plus a slow-rotating spiral arm of dim stars.
- **Glow**: draw the shape twice — once large with low alpha and `globalCompositeOperation = 'lighter'`, once normal on top. Gives the "shiny cosmic retro" feel cheaply.
- **Scanline overlay** (optional, looks great): every 2nd row tinted slightly darker as a full-screen pass.

### 3d. Color palette
Pick one cohesive retro palette and reference it everywhere. Suggested:

```
0: transparent
1: #0a0420  (deep space)
2: #2a1058  (purple void)
3: #ff3ec9  (magenta neon)
4: #00f0ff  (cyan neon)
5: #ffe66d  (star yellow)
6: #ff4444  (danger red)
7: #ffffff  (pure white)
8: #6effb0  (mint accent)
```

---

## 4. Game States (state machine)

```
MENU  →  PLAYING  →  BOSS  →  CREDITS
                ↓        ↓
              DEATH ←————┘
                ↓
              MENU
```

- **MENU**: title + "PRESS SPACE TO PLAY" + small credits line at bottom (`www  ·  AugustoCorre  ·  Loading...`). Animated parallax stars behind. Title bobs / rotates slightly.
- **PLAYING**: regular wave-based shooter. Score counter top-right. Lives top-left. Powerup status icons.
- **BOSS**: triggered when `score >= 3000`. Regular spawns stop, screen shakes briefly, boss enters from top.
- **DEATH**: explosion plays out (~1.5s), then "GAME OVER — PRESS SPACE" → returns to MENU.
- **CREDITS**: triggered after boss defeat. Slow vertical scroll of the same three credit lines, on the cosmic background, with a "PRESS SPACE TO RESTART" at the end.

---

## 5. Controls

| Key | Action |
|---|---|
| `W` `A` `S` `D` | Move player ship (8-directional) |
| `Space` | Shoot (also: start game from menu, restart from credits/death) |
| `Esc` | Pause / unpause during PLAYING and BOSS |

Pause: dim the screen, draw "PAUSED" centered, freeze updates but keep render running.

---

## 6. Player

- Small voxel ship (~6 wide × 4 tall × 3 deep voxels). Cyan + magenta accent.
- Movement: 8-directional, clamped to screen.
- Default fire rate: ~5 bullets/sec. Bullets are short magenta rectangles with a cyan core and additive glow.
- 3 lives. On hit: brief invincibility flash + lose 1 life. On 0 lives: trigger DEATH state.
- A small thruster particle trail behind the ship (orange/yellow particles, additive blend, fade out).

---

## 7. Enemies

Enemies spawn from the top of the screen, move down with type-specific patterns. Each kill = **+10 points**.

**Wave difficulty**: spawn rate scales with elapsed time. Suggested curve:

```
spawnInterval = max(0.3s, 1.6s - elapsedSeconds * 0.015)
```

So you start with ~1 enemy/1.6s and ramp toward ~3 enemies/sec by the time the player approaches 3000 points.

### Enemy types (3 + boss)

1. **Grunt alien** — small pixel-sprite, straight-down movement, doesn't shoot. Easy filler.
2. **Big Dog** — larger pixel sprite (~16×12), bobs side-to-side as it descends, occasionally barks (drops a bone-shaped projectile). Pixel art: square head, triangle ears, tongue out, derpy eyes.
3. **Giant Piano** — wide sprite (~20×10) with black-and-white key pattern. Floats slowly, shoots musical-note bullets in a spread. Looks unhinged.

(Feel free to add more "crazy" enemies in the same vein — a flying toaster, a sentient sandwich — but the spec only requires dog + piano so start with those.)

> **Open question:** the spec says "crazy enemies" plural with two named. Plan delivers **3 regular enemy types** (grunt + dog + piano). Tell me if you want more designed up front.

---

## 8. Powerups

Drop randomly from killed enemies (~8% chance). Each is a small floating pixel icon with a colored glow halo, drifting slowly downward. Picked up by overlap with the player. Effects last 10 seconds (except shield, which absorbs 1 hit, and bomb, which is instant).

| # | Name | Icon hint | Effect |
|---|---|---|---|
| 1 | **Rapid Fire** | yellow lightning bolt | 2× fire rate |
| 2 | **Triple Shot** | three magenta dots | fires 3 bullets in a fan |
| 3 | **Armor / Shield** | cyan ring | absorbs 1 hit; visible halo around player |
| 4 | **Speed Boost** | green chevron | 1.5× movement speed |
| 5 | **Cosmic Bomb** | white star | instantly clears every on-screen non-boss enemy with a screen-wide flash |

> **Open question:** spec named "shooting" and "armor" specifically. The other 3 are my picks — swap them if you have something else in mind.

---

## 9. Boss: The Cosmic Clock

Triggers at score ≥ 3000. **Drawn entirely with vector primitives** (no sprite). Specs:

- **Size**: ~60% of screen height.
- **Position**: enters from top, settles at upper-center. Slow vertical bob.
- **Visual layers** (back to front):
  1. Outer glow halo (large radial gradient, additive blend, magenta).
  2. Outer ring of zodiac/runic ticks: 12 small triangles around the perimeter.
  3. Clock face: dark purple disc with cyan rim (stroked circle).
  4. **Roman numerals I–XII**, drawn as line segments (no font dependency — define each numeral as a small vector path).
  5. Three rotating hands: hour (slow), minute (medium), second (fast — this one shoots laser beams at the player on each "tick").
  6. Central pivot: glowing star.
  7. Optional: faint orbiting moons around the clock at different radii.
- **Behavior**:
  - Health: enough to take ~30–60 hits.
  - Phase 1 (>50% HP): second-hand fires a beam every 2s.
  - Phase 2 (<50% HP): also spawns "minute" projectiles from numeral positions in radial bursts.
  - On death: violent screen shake, large multi-stage explosion (see §11), 1s pause on black, then CREDITS state.

Hand drawing: just rotated rectangles with rounded ends, neon-colored, with additive glow.

---

## 10. Title Rendering — "SPACE YESTERDAY 8000" in 3D Voxels

Each character defined as a small voxel grid. Two reasonable widths:

- **5w × 7h × 2d voxels** per glyph (cheap, readable, classic arcade feel). Recommended.
- 7w × 9h × 3d if you want chunkier letters.

Steps:

1. Define a font map: object keyed by character → 3D array of 0/1 (filled voxels).
2. Need glyphs for: `S P A C E Y T R D 8 0`. (Letters in "SPACE YESTERDAY 8000" — note: no `1`, etc., needed.)
3. Render the whole string by laying glyphs left-to-right with 1-voxel spacing.
4. Apply a **slight tilt + slow Y-axis sway** by recomputing the iso projection per frame with a rotating offset, OR keep it static with only a vertical bob — either reads as 3D.
5. Fill voxels with a **vertical gradient palette**: top voxels magenta, middle cyan, bottom purple. Each cube gets 3 shades of its base color for the iso faces.
6. Add a soft additive glow pass underneath.

The title appears: large on MENU, smaller version optional during CREDITS.

---

## 11. Death Explosion (drawn with code)

Two combined effects:

1. **Particle burst**: ~80 particles emitted from the player's last position. Each has random velocity, color from {yellow, orange, magenta, white}, size 1–3 pixels, lifetime 0.5–1.5s, rendered additive. Velocity damped over time, slight gravity.
2. **Shockwave ring**: an expanding stroked circle, alpha fading from 1 → 0 over 0.6s, line width shrinking, color cyan.
3. **Screen flash**: 1 frame of full white at alpha 0.6, then fade out over 0.2s.
4. **Screen shake**: random ±4px offset for 0.4s.

The **boss death** uses the same system but bigger, longer (~2s), and emits 4–5 sequential bursts before the final flash.

---

## 12. Credits Screen

Triggered after boss defeat. Layout:

```
              SPACE YESTERDAY 8000
                  — CREDITS —

                       www
                  AugustoCorre
                   Loading...

           PRESS SPACE TO PLAY AGAIN
```

- Each line fades in sequentially, ~0.6s apart.
- Small slow-scroll upward.
- Stars + nebula still animating in the back.
- Same three credit lines also appear small at the bottom of the MENU.

---

## 13. HUD

Top bar, 1px line below it for a retro divider:

- **Top-left**: `LIVES: ♥ ♥ ♥` (hearts drawn as tiny pixel sprites).
- **Top-center**: progress bar to boss → `[████████░░] 1840 / 3000`. Hides when boss is active; replaced with boss HP bar.
- **Top-right**: `SCORE: 0000`.
- **Bottom-left**: row of active powerup icons with shrinking timer rings.

---

## 14. Implementation Phases (recommended order for Claude Code)

1. **Skeleton**: `index.html`, canvas setup, fixed-timestep loop, palette, input handler. Clear screen to deep-space color.
2. **Background**: 3 parallax star layers + nebula gradients. This establishes the "shiny cosmic retro" tone immediately.
3. **Render helpers**: pixel-sprite drawer, voxel cube + voxel model drawer. Test by rendering a placeholder voxel cube center-screen.
4. **Title**: voxel font, render "SPACE YESTERDAY 8000". Build the MENU state.
5. **Player**: voxel ship + WASD movement + bullets + thruster particles.
6. **Enemies**: spawn system, grunt first, then dog, then piano. Hit detection, score increment.
7. **Death + explosion system**: kills the player, plays explosion, returns to MENU.
8. **Powerups**: drops, pickup, 5 effects. Powerup HUD icons.
9. **Boss trigger + cosmic clock**: drawn vector clock, two phases, attacks, HP bar.
10. **Boss death + credits state** with scrolling credits.
11. **Pause** (Esc) overlay.
12. **Polish pass**: scanlines, screen shake, bloom-ish glow tuning, small-screen scaling, color tweaks.

Each phase should be independently runnable so you can verify visuals before moving on.

---

## 15. Quality Targets

- Runs at solid 60fps on a mid-range laptop, even with ~30 enemies + particles on screen.
- No text rendered with browser system fonts — every letter is a voxel glyph or a small pixel-grid glyph (define a tiny 3×5 pixel font for HUD numbers and the menu instructions). This keeps the aesthetic consistent.
- Crisp pixel scaling: the canvas's logical size stays small (~480×270), CSS-scales to fill window with `image-rendering: pixelated`.
- Total file size: under ~80 KB minified. Everything fits in one HTML file.

---

## 16. Things Intentionally Out of Scope (unless you say otherwise)

- Audio / music
- Mobile touch controls
- Persistent high scores (localStorage)
- Multiple bosses
- Settings menu

---

## Summary of Open Questions

1. **Audio**: silent, or chiptune via WebAudio?
2. **Number of regular enemy types**: 3 (grunt + dog + piano) is what I've planned. More?
3. **Powerups 3, 4, 5**: I picked Rapid Fire, Triple Shot, Speed Boost, Bomb (with Armor as #3 from your spec). Replace any?
4. **HUD font**: tiny pixel-grid glyphs, OR voxel-style for numbers too? (Voxel for numbers is more on-theme but harder to read at small sizes.)

Default answers if you don't reply: silent, 3 enemies, those powerups, tiny pixel font for HUD.
