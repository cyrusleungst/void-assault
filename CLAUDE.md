# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git discipline

After completing any meaningful unit of work — a new feature, a bug fix, a refactor — commit and push immediately. Never leave the repo in a state where work exists locally but not on GitHub.

```bash
git add <files>
git commit -m "Short description of what changed and why"
git push
```

Commit messages should be specific (e.g. `"Fix enemy spawning off wrong edge"`, `"Add boss enemy type to level 5"`), not vague (`"updates"`, `"fix stuff"`). Each commit should represent one logical change so it's easy to revert a specific piece of work without undoing unrelated changes.

## Running the game

ES modules require HTTP — open via a local server, not directly from disk:

```bash
cd shooter && python3 -m http.server 8080
# open http://localhost:8080
```

There are no build steps, bundlers, or package managers.

## Architecture

The game is in `shooter/` and written as vanilla ES modules loaded by `index.html`. `main.js` is the entry point — it creates a `Game` instance and calls `game.start()`.

**`game.js` is the central orchestrator.** It owns the state machine, the `requestAnimationFrame` loop, and all entity arrays (`bullets`, `enemies`). Every other module is called from here. The loop dispatches to a `_tick*` method per state (MENU, PLAYING, LEVEL_COMPLETE, GAME_OVER, PAUSED), each of which handles both update and draw for that state.

**Data flows one way:** `InputHandler` → `Player`/`Game` → entity arrays → `_checkCollisions()` → HUD/UI draw. Nothing outside `game.js` holds references to sibling modules.

**`constants.js` is the single source of truth** for all tunable values: canvas size, player stats, bullet damage, enemy definitions (`ENEMY_DEFS`), and the full level config (`LEVELS` array). Balancing changes start and end here.

**Level progression** is managed by `LevelManager` (`level.js`). It reads wave configs from `LEVELS`, tracks spawn timers, and sets `levelComplete`/`gameWon` flags that `game.js` polls each frame. It does not hold references to `game.js`.

**Sprites are drawn programmatically** using Canvas 2D API primitives — no image files. Each entity class has its own `draw(ctx)` method. Enemy types (grunt, tank, speeder) are selected at construction time from `ENEMY_DEFS` and rendered with type-specific `_draw*` methods in `Enemy`.

**Particles** (`particle.js`) are split into two layers: `underlayParticles` (explosions, hit sparks — drawn before sprites) and `overlayParticles` (muzzle flash — drawn after the player). `game.js` calls `drawUnder` and `drawOver` at the appropriate points in the render order.

**Input** uses a `keys` presence map plus a `_justPressed` map that is cleared each frame via `endFrame()`. Use `isJustPressed()` for one-shot actions (Escape, Enter) and `isDown()` only for held actions (movement, shooting).

**Dead-object cleanup**: entities mark themselves `dead = true`; `game.js` filters both arrays each frame with `.filter(x => !x.dead)`.

**`dt` is capped at 100ms** (`Math.min(rawDt, 0.1)`) to prevent large jumps when the tab is backgrounded and brought back.
