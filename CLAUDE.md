# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the game

Open `index.html` directly in a browser — no build step, no server, no dependencies to install. The only external resource is the Press Start 2P font loaded from Google Fonts (requires internet; falls back to monospace if offline).

## Architecture

The entire game lives in a single `index.html` file with inline CSS and JavaScript. There is no module system or bundler.

### Game states

The variable `gameState` drives everything. Valid values: `MENU` → `PLAYING` ↔ `LEVEL_UP` → `GAME_OVER`. The `update()` function returns early for any state other than `PLAYING`.

`paused` (boolean) is a separate flag that gates `update()` without changing `gameState`. It is toggled by the `P` key or the on-screen PAUSE button and is reset to `false` on game reset.

Wave transitions are seamless — when `enemies.length === 0`, the next wave spawns immediately and a `waveBanner` object (`{ alpha, line1, line2 }`) is set to fade out over 1.5 s. There is no `WAVE_CLEAR` pause state.

### Coordinate systems

Two coordinate spaces exist simultaneously:
- **World space** (2400×2400): where all entities live (`player.x/y`, `enemies`, `bullets`, etc.)
- **Screen space** (900×600): the visible canvas

The `cam` object (`cam.x`, `cam.y`) is the top-left corner of the viewport in world space. All world-space drawing is wrapped in `ctx.save() / ctx.translate(-cam.x, -cam.y) / ctx.restore()`. Mouse position (`mouse.x/y`) is in screen space and must be converted to world space by adding `cam.x/y` before use.

### Entity collections

Six mutable arrays hold live entities: `bullets`, `enemies`, `gems`, `powerups`, `particles`, `activeEffects`. Each frame, dead/expired items are filtered out at the end of their respective update block.

### Upgrade & powerup systems

- `UPGRADE_POOL` (12 entries) — permanent stat changes applied to `player` at level-up. `triggerLevelUp()` splices 3 random entries from a copy of the pool and pauses the game.
- `POWERUP_TYPES` (4 entries) — temporary or instant pickups that drop from enemies with 13% probability. Timed buffs use `addTimedBuff()` which pushes an entry into `activeEffects` with an `onRemove` callback.

### Pause button

`PAUSE_BTN = { x: W/2 - 30, y: 8, w: 60, h: 28 }` is a constant defining the clickable/tappable pause button centered at the top of the canvas. It is drawn inside `drawHUD()` only when `gameState === 'PLAYING'`. Click, touch, and `P` key all toggle `paused`. The button label switches between `PAUSE` and `RESUME`.

### Audio system

All sound is synthesized at runtime via the Web Audio API — no audio files. `audioCtx` is created lazily on first interaction to satisfy browser autoplay policy. `playTone()` creates oscillator nodes; `playNoise()` creates white noise via an `AudioBuffer`. High-frequency sounds (`sfxShoot`, `sfxHit`, `sfxPlayerHurt`) are throttled with timestamp guards to prevent audio clipping when many events fire per frame. The `muted` boolean gates all sound output; toggled with `M`.

### Wave scaling

`getWaveComposition(w)` returns counts per enemy type. Every 5th wave is a boss wave (`w % 5 === 0`) that includes one boss plus extra grunts and scouts. Enemy health/count scales linearly with wave number. Boss `health` is computed at spawn time using the current `wave` variable.

### Touch / mobile controls

`touchState` holds two slots: `joy` (left-half joystick) and `aim` (right-half shoot target). Both are `null` when inactive. Touch events are wired on the canvas with `{ passive: false }` so `e.preventDefault()` can suppress browser scrolling/zooming.

`getTouchPos()` applies a scale factor (`canvas internal size / CSS rendered size`) so touch coordinates map correctly to game coordinates even when the canvas is CSS-scaled down on small screens. The joystick contributes additively to `dx/dy` in the movement block — keyboard and touch movement can overlap safely.

The canvas uses `touch-action: none` (CSS) to block default gestures, and a `@media` query scales the canvas to full viewport width on screens narrower than 900px while preserving the 900×600 aspect ratio.

### Rendering order

Back to front within the world-space transform: particles → gems → powerups → enemies (with health bars) → bullets → player. HUD and overlay screens (menu, level-up, wave clear, game over) are drawn after `ctx.restore()` in screen space so they are never affected by the camera transform.
