---
name: web-game-dev
description: Build polished browser games — game loop, input, collision, state machines, juice/feel, audio, and performance — in Canvas 2D, DOM, or lightweight engines (Phaser, Kaplay). Use this skill WHENEVER the user wants to create any game, playable demo, interactive toy, arcade clone, puzzle, platformer, or gamified learning tool. Trigger on "משחק", "משחקון", "game", "arcade", "פלטפורמר", "snake/tetris/pong/flappy clone", "something playable", or any request where the output is meant to be PLAYED. Don't build games as plain event-handler spaghetti — always use this skill's architecture.
---

# Web Game Development

A game is a real-time simulation, not a form with pictures. The difference between "works" and "feels great" is architecture + juice, and both are cheap if applied from the start.

## Tech Choice (decide first, tell the user why)

- **Canvas 2D + vanilla JS**: default for arcade/puzzle/single-screen games, and for Claude artifacts. Zero deps, full control, teaches fundamentals.
- **DOM/CSS**: card games, board games, word games — anything grid/turn-based where CSS transitions do the animation. Don't use DOM for 60fps bullet-hell.
- **Phaser / Kaplay**: real projects needing physics, tilemaps, sprite atlases, scenes. Don't hand-roll a physics engine for a platformer with slopes.
- **Three.js**: 3D only. In claude.ai artifacts, verify which Three.js version the environment provides before using newer APIs (historically pinned to an older release without OrbitControls/CapsuleGeometry).

## The Loop: Fixed Timestep, Variable Render

Never simulate with raw per-frame deltas tied to refresh rate (breaks on 144Hz monitors) and never `setInterval`. The canonical loop:

```js
const STEP = 1000 / 60; // simulate at fixed 60Hz
let acc = 0, last = performance.now();

function frame(now) {
  acc += Math.min(now - last, 250); // clamp: tab-switch spiral-of-death guard
  last = now;
  while (acc >= STEP) { update(STEP / 1000); acc -= STEP; }
  render(acc / STEP); // optional interpolation factor
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

- `update(dt)` mutates state; `render()` only draws. Never mix — this separation is what makes pause, replay, and speed-up trivial.
- All movement in units/second scaled by dt: `x += speed * dt`, never `x += speed`.
- Pause on `visibilitychange` (also pauses audio).

## Architecture: State Machines Everywhere

Two levels, both explicit:

1. **Game states**: `TITLE → PLAYING → PAUSED → GAME_OVER`. A single `state` variable with per-state update/render/input. No `if (!gameOver && !paused && started)` pyramids.
2. **Entity states**: player is `idle | running | jumping | hurt | dead`. Transitions in one place. This is where "can't jump while dead" bugs die.

Entities: plain objects in arrays `{pos, vel, size, hp, state}` updated by systems (a light ECS flavor). Removal: mark `dead` and filter after the update pass — splicing mid-iteration skips entities.

```js
entities = entities.filter(e => !e.dead); // after update, never during
```

## Input: Sampled, Not Event-Driven

Events set flags; the update loop reads them. Gameplay logic inside keydown handlers causes missed/duplicate actions and OS key-repeat bugs.

```js
const keys = {};
addEventListener('keydown', e => { keys[e.code] = true; });
addEventListener('keyup',   e => { keys[e.code] = false; });
// in update: if (keys.Space && player.grounded) jump();
```

- Use `e.code` (layout-independent — matters for Hebrew keyboards).
- **Mobile is not optional**: add touch controls (on-screen buttons / swipe / tap zones) and `touch-action: none` on the canvas. Test tap targets ≥ 48px.
- Buffer jump inputs (~100ms) and add coyote time (~80ms after leaving a ledge) — platformers feel broken without these two.

## Collision

- Circles: `dist² < (r1+r2)²` (skip the sqrt). AABB: standard overlap test. This covers 90% of 2D games.
- Fast objects tunnel through thin walls at fixed-step too — if bullets are fast, raycast/step the movement.
- >50 entities checking each other: spatial hash grid before optimizing anything else.
- Resolve collisions axis-by-axis (move X, resolve, move Y, resolve) — fixes "sticking to walls" in platformers.

## Juice — The Section That Makes It Feel Good

Budget 20% of dev time here; it's the difference players actually perceive. In rough order of impact-per-line:

1. **Tweens/easing** on everything that moves UI-wise; nothing snaps. `easeOutCubic` is the workhorse: `1 - (1-t)**3`.
2. **Squash & stretch**: scale 1.2x/0.8x on jump, invert on land, spring back.
3. **Screen shake** on impact: offset camera by decaying random vector (magnitude 4–8px, 150ms). Add a reduce-motion toggle.
4. **Hit-stop**: freeze the sim 40–80ms on significant hits.
5. **Particles**: one pooled system (spawn/update/draw with gravity + fade) reused for dust, explosions, pickups, trails.
6. **Flash + knockback** on damage (white flash 2 frames), floating score numbers, screen flash on level-up.
7. **Sound on every interaction** — see audio below. A silent game feels broken.

## Audio

- WebAudio API, and generate SFX procedurally when assets are a hassle (great for artifacts): a short oscillator with pitch/gain envelopes covers jump/coin/hit/explosion (jsfxr-style).
- Browsers block audio before user gesture: create/resume `AudioContext` on first input, show a mute toggle, persist the preference.
- Pool/reuse buffers; never `new Audio()` per shot.

## Performance

- Never allocate in the hot loop (no object/array literals per frame in update/render paths) — GC pauses read as stutter. Pool bullets/particles.
- Draw order: clear once, batch by style, avoid `save()/restore()` per entity, round pixel coords for crisp sprites.
- Offscreen canvas for static layers (background, tile map) — draw once, blit per frame.
- `image-rendering: pixelated` for pixel-art scaling.
- Target: 60fps with headroom; profile with the Performance tab before optimizing blind.

## Game Design Minimums (ship-quality checklist)

1. Instant restart (R / tap) from game over — friction here kills replay
2. Difficulty ramps (speed/spawn-rate curve), not static
3. Score + persistent best score (localStorage when available; in-memory in artifacts — say so)
4. Clear feedback for every player action within 100ms
5. Title screen with one-line controls; playable within 2 seconds of load
6. Fail state is legible — player always knows WHY they died
7. Mobile-playable, RTL-safe UI text if Hebrew
8. Deterministic sim (seedable RNG) if replays/daily-challenge are ever wanted — costs nothing now, impossible to retrofit

## Common AI-Generated Game Bugs — Check Explicitly

- Speed tied to frame rate (missing dt) — test mentally at 144Hz
- Input handled in event listeners with gameplay side effects
- Entities spliced during iteration
- Collision checked before movement resolved, or once per pair per axis
- Restart that doesn't fully reset state (lingering timers, listeners, velocities)
- `setInterval` timers desyncing from the loop
- Memory leak: listeners re-registered on every restart
