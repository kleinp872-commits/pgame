# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`mo-shou_3.html` — 墨守 · **Ink Guard**, a single-file HTML5 canvas game. Chinese
ink-wash (水墨) themed tower-defense/slasher: ink spirits (墨靈) drift down the
screen and the player swipes/traces to slay them before they cross the red seal
line. Entirely self-contained: markup, CSS `:root` theme tokens, and the game
engine (one IIFE `<script>`) all live in that one file. No build, no dependencies,
no tests, no package manager. To run/dev, open the file in a browser (mobile
viewport — it's touch-first, `maximum-scale=1.0, user-scalable=no`).

## Architecture (all in the `<script>` IIFE, top-to-bottom)

- **Three stacked canvases.** `#bg` is drawn once per `resize()` (static ink-wash
  mountains + ripple lines + procedural paper texture + the dashed red defense
  line at `baseY()` = `H*0.82`). `#stain` holds persistent kill-stains, faded by
  a throttled `destination-out` pass every 2s. `#game` is cleared and redrawn
  every frame. `DPR` is capped at 2; contexts are pre-transformed by
  `setTransform(DPR,...)` so all game logic uses CSS pixels.
- **Module zones in the IIFE** (in order): `audio` (Web Audio synthesized `sfx.*`
  + mute persisted to localStorage), `persist` (best score), paper/stain/shards/
  lightning/fog/ink-wave fx, floaters + nameplate HUD helpers, `juice`
  (`shake()`, `buzz()`, hit-stop via `freezeT`), then the original game core.
- **Ring object pools.** `particles` (256) / `floaters` (32) / `shards` (64) are
  fixed-size `const` arrays with `alive` flags and cursor-based spawn — never
  reassign or `filter()` them; reset via `alive=false` sweep in `resetGame()`.
- **Per-frame split:** `updateFx(dt)` runs always (even during hit-stop freeze
  and level transition); game logic in `update(dt)` early-returns on `freezeT`
  and during `levelTransition`.
- **Cast patterns.** `CAST_PATTERNS` (W/V/Z/O) each own a `gen()` for the dashed
  guide and a `match(strokeStats)` heuristic; `checkCastMatch` only tests the
  active event's own pattern.
- **Fixed game-state module-locals** (`score, lives, level, hp, enemies,
  particles, strokePts, strokeTrail, inkBlooms, castEvent, ...`). `resetGame()`
  reinitializes them; there is no other state container.
- **Main loop** `loop(t)` → `update(dt)` + `render()`, driven by
  `requestAnimationFrame`. `dt` is clamped to 50 ms. `update` only runs when
  `running && !paused`; `render` always runs.
- **Data-driven levels.** `LEVELS[]` (1-indexed; index 0 is `null`) declares which
  animals spawn, their per-type stroke `hits`, spawn `weight`, level `goal`, and
  `spawnInterval`. `getLevelConfig(lvl)` extrapolates past the last defined level
  (adds hits/goal, shrinks interval). `ANIMAL_DEFS` holds per-type visual/motion
  constants (`amp`, radius range, `speedMult`). To add an enemy: add an
  `ANIMAL_DEFS` entry + a `drawEnemy` branch + reference it in `LEVELS`.
- **Enemies** are plain objects pushed by `spawnEnemy()`. Motion = downward
  `speed` (px/ms) plus horizontal sine-swim around `baseX`. `drawEnemy` switches on
  `e.type` (`tadpole`/`octopus`/`jellyfish`/`koi`/`boss`), all hand-drawn with
  canvas paths. Multi-hit enemies show remaining-hit dots above them (boss draws
  an HP bar instead).
- **Friendly koi.** From level 2 on, `spawnEnemy` has a ~9% chance to spawn a
  golden `koi` with `friendly:true`. Slashing it calls `hitFriendly` (−10 HP,
  −30 score, combo reset); it crosses `baseY()` penalty-free and thunder skips
  `friendly` enemies.
- **Final boss.** Clearing `FINAL_LEVEL` (10) calls `startBossPhase()` instead of
  `startLevel(11)`: pushes a `boss` (`BOSS_HP` hits) into `enemies` plus sparse
  tadpole minions (minion kills don't advance `levelKills`). Boss death →
  `victory()` (win overlay); boss crossing the seal line → instant `gameOver()`.
- **Input → combat.** Pointer/touch handlers build `strokePts` (with timestamps).
  On `pointermove`, each new segment is collision-tested against every enemy via
  `distToSeg`; a hit calls `hitEnemy` (respects `hitCooldown`, increments `hits`,
  kills at `maxHits`). `strokeTrail` is the fading brush-ink visual.
- **"Trace to cast" special.** `triggerCastEvent()` shows a dashed zig-zag path;
  `checkCastMatch()` is a heuristic (counts x-direction reversals + proximity +
  path length) — matching it fires `resolveCast(true)`: thunder that deals
  `THUNDER_DMG` (2) damage per enemy via `thunderStrike` (boss takes
  `BOSS_THUNDER_DMG`), so tough spirits survive and need several casts. Governed
  by `castCooldown` (14–30 s random).
- **Effects.** `inkBloom()` (radial-gradient ink splashes with animated tendrils —
  the signature visual) and `burst()`/`particles` are purely cosmetic.
- **HUD** is DOM, not canvas: `updateHUD()` writes score/level/hp/lives/wave-bar.
  Overlays (`#startOverlay`, `#endOverlay`) toggle the `.show` class.

## Conventions specific to this file

- **Units:** all gameplay math is in CSS pixels; speeds are **px per millisecond**
  and scaled by `dt`. Don't reintroduce per-frame (frame-rate-dependent) movement.
- **Losing:** an enemy crossing `baseY()` costs a life *and* 14 HP; `gameOver()`
  fires when either hits 0.
- **Colors** come from the CSS `:root` palette (paper/ink/seal). Canvas code uses
  matching literal `rgba()` strings — the seal-red `161,58,44` recurs for "danger"
  blooms (enemy breach, level-clear, thunder).
- Language mix is intentional: Traditional/Simplified Chinese in UI copy, English
  in the flash/prompt strings.
