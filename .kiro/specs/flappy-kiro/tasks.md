# Implementation Plan: Flappy Kiro

## Overview

Build Flappy Kiro as a single, dependency-free `index.html` (plus the existing `assets/` folder) that runs directly via `file://` with plain HTML/CSS/JS and an HTML5 canvas — no frameworks, no build tools. The plan builds incrementally: first the frozen `Config` object and canvas bootstrap, then the pure logic core (physics, geometry/hitbox, obstacles, coins, collision/boundaries, scoring, difficulty, chart), then the IO layer (input handler with FSM routing, renderer, audio manager, storage, canvas scaler), then wiring the delta-time game loop and the three-state FSM, and finally the three screens, HUD, and trading/finance visual polish. Each step builds on the previous and ends by integrating into the running game so there is no orphaned code.

**ALL testing tasks are optional** (marked with `*`) per the design's Testing Strategy. No game-building task depends on any test task, and tests never block or gate the build. Optional property tests use fast-check in a dev-only Node harness that imports the pure core; the shipped `index.html` stays self-contained.

## Tasks

- [ ] 1. Bootstrap single-file game shell and Config
  - [ ] 1.1 Create `index.html` scaffold with inline `<style>` and `<script>`, a `<canvas>` at logical resolution 800×600, dark page background, and no external URLs (local relative references only)
    - Set the canvas backing store to 800×600; center it in the viewport
    - Verify it opens cleanly under `file://` with no console errors
    - _Requirements: 1.1, 1.2, 13.5_
  - [ ] 1.2 Define the frozen `Config` object with all tunable constants and the `palette`
    - Include gravity, jumpImpulse, maxDeltaTime, player ratios, hitboxScale, spriteWidth/spriteHeight (48), initialSpeed, speedIncrement, maxSpeed, volatilityStep, initialGap, gapReduction, minGap, maxGapDelta (180), obstacleWidth, wickWidth, spawnInterval, coinBonus, coinSpawnChance, coinRadius, restartLockoutMs (500), minFontSize, and `palette` (bg/up/down/text/gold)
    - Express all velocity/acceleration values in per-second units; `Object.freeze` the object
    - _Requirements: 1.3, 3.6, 6.6, 10.3, 14.2_

- [ ] 2. Implement physics and player geometry (pure logic core)
  - [ ] 2.1 Implement `integratePlayer(player, dt, config)` and `applyJump(player, config)`
    - Gravity: `vy = vy + gravity*dt`; position: `y = y + vy*dt`; jump sets `vy = -jumpImpulse` unconditionally
    - Pure functions returning new `{ y, vy }`; no DOM or global reads
    - _Requirements: 3.2, 3.3, 3.4, 14.1, 14.2 / Design: Physics_
  - [ ]* 2.2 Write property test for jump impulse
    - **Property 1: Jump applies the configured upward impulse**
    - **Validates: Requirements 3.3, 3.4**
  - [ ]* 2.3 Write property test for gravity integration
    - **Property 2: Gravity integration**
    - **Validates: Requirements 3.2, 14.1, 14.2**
  - [ ] 2.4 Implement derived horizontal position and `playerHitbox(player, config)`
    - Ghosty x = `playerXRatio * canvasWidth`; hitbox = `0.8*spriteWidth × 0.8*spriteHeight`, centered on the Config-defined rendered sprite size
    - _Requirements: 3.1, 3.5, 3.7 / Design: Hitbox / Geometry_
  - [ ]* 2.5 Write property test for invariant horizontal position
    - **Property 3: Horizontal position is invariant**
    - **Validates: Requirements 3.5**
  - [ ]* 2.6 Write property test for hitbox dimensions
    - **Property 4: Hitbox is 80% of the Config-defined rendered sprite size, centered**
    - **Validates: Requirements 3.7**

- [ ] 3. Implement obstacle system (pure logic core)
  - [ ] 3.1 Implement `spawnObstacle(config, gapSize, prevGapCenterY, rng)`
    - Choose gap center via injected rng; constrain to within `maxGapDelta` of `prevGapCenterY`, then clamp to `[gapSize/2, canvasHeight - gapSize/2]`; return `Obstacle` with both derived pillar rects, `passed=false`, `coin=null`
    - _Requirements: 4.1, 4.5, 4.6 / Design: Obstacle System_
  - [ ] 3.2 Implement `updateObstacles(obstacles, dt, speed)` and `shouldSpawn(elapsed, config)`
    - Advance each obstacle left by `speed*dt`; cull any with right edge (`x+width`) ≤ 0; `shouldSpawn` true iff `elapsed >= spawnInterval`
    - _Requirements: 4.3, 4.4, 4.7 / Design: Obstacle System_
  - [ ]* 3.3 Write property test for obstacle scrolling
    - **Property 5: Obstacles scroll left by speed × dt**
    - **Validates: Requirements 4.3**
  - [ ]* 3.4 Write property test for obstacle culling
    - **Property 6: Off-screen obstacles are culled**
    - **Validates: Requirements 4.4**
  - [ ]* 3.5 Write property test for on-canvas gap center
    - **Property 7: Spawned gap center keeps both pillars on-canvas**
    - **Validates: Requirements 4.6**
  - [ ]* 3.6 Write property test for spawn interval threshold
    - **Property 8: Obstacle spawn interval threshold**
    - **Validates: Requirements 4.7**
  - [ ]* 3.7 Write property test for reachable consecutive gap centers
    - **Property 30: Consecutive gap centers are reachable**
    - **Validates: Requirements 4.6**

- [ ] 4. Implement coin system (pure logic core)
  - [ ] 4.1 Implement `maybeSpawnCoin(obstacle, config, rng)` and `tryCollectCoin(playerBox, coin)`
    - Spawn a coin iff `rng < coinSpawnChance`, positioned at the gap's horizontal and vertical midpoint; collect when the player hitbox overlaps the coin, mark collected so it cannot be recollected
    - _Requirements: 6.1, 6.2, 6.3, 6.4 / Design: Coin System_
  - [ ]* 4.2 Write property test for coin spawn chance
    - **Property 10: Coin spawns according to the chance threshold**
    - **Validates: Requirements 6.1**
  - [ ]* 4.3 Write property test for coin centering
    - **Property 11: Coin is centered in the gap**
    - **Validates: Requirements 6.2**
  - [ ]* 4.4 Write property test for coin collection and removal
    - **Property 12: Overlapping coin is collected, bonus added, coin removed**
    - **Validates: Requirements 6.3, 6.4**
  - [ ]* 4.5 Write property test for culled coin removal
    - **Property 13: A culled obstacle takes its coin with it**
    - **Validates: Requirements 6.5**

- [ ] 5. Implement collision, boundaries, and scoring (pure logic core)
  - [ ] 5.1 Implement `aabbIntersect(a, b)`, `hitsAnyObstacle(playerBox, obstacles)`, and `boundaryCross(playerBox, canvasHeight)`
    - AABB overlap test; obstacle collision against either pillar of any obstacle; boundary returns `'top' | 'bottom' | null`
    - _Requirements: 7.1, 7.2, 7.3 / Design: Collision & Boundaries_
  - [ ] 5.2 Implement `scoreGapPasses(profit, obstacles, playerX)` and `addCoinBonus(profit, config)`
    - Increment profit by 1 per obstacle whose gap right edge the player crossed and that is not yet `passed`; idempotent on already-counted obstacles; coin bonus adds `coinBonus`
    - _Requirements: 5.1, 6.3, 6.6 / Design: Scoring_
  - [ ]* 5.3 Write property test for obstacle collision exactness
    - **Property 14: Obstacle collision detection is exact**
    - **Validates: Requirements 7.1**
  - [ ]* 5.4 Write property test for boundary crossing
    - **Property 15: Boundary crossing detection**
    - **Validates: Requirements 7.2, 7.3**
  - [ ]* 5.5 Write property test for score-once-per-gap idempotence
    - **Property 9: Each passed gap scores exactly once**
    - **Validates: Requirements 5.1**

- [ ] 6. Implement difficulty scaling (pure logic core)
  - [ ] 6.1 Implement `difficultyLevel(profit, config)`, `speedForLevel(level, config)`, and `gapForLevel(level, config)`
    - Level = `floor(profit / volatilityStep)`; speed = `min(initialSpeed + level*speedIncrement, maxSpeed)`; gap = `max(initialGap - level*gapReduction, minGap)` — closed-form so no level is skipped on a coin-driven jump
    - _Requirements: 10.1, 10.2, 10.3 / Design: Difficulty_
  - [ ]* 6.2 Write property test for difficulty level formula
    - **Property 21: Difficulty level formula**
    - **Validates: Requirements 10.1**
  - [ ]* 6.3 Write property test for clamped closed-form speed/gap
    - **Property 22: Speed and gap are clamped closed-form functions of level (no level skipped)**
    - **Validates: Requirements 10.2**

- [ ] 7. Implement chart generator (pure logic core)
  - [ ] 7.1 Implement `updateChart(segments, dt, speed, level, config, rng)`
    - Scroll segments left by `speed*dt` (same rate as obstacles); append new procedural segments so `[0, canvasWidth]` is always covered with no gap/repeat; cull segments fully left of x=0; scale amplitude/jaggedness with `level`
    - _Requirements: 11.2, 11.3, 11.4, 13.7 / Design: Chart Generator_
  - [ ]* 7.2 Write property test for chart scroll rate
    - **Property 23: Chart scrolls at the obstacle speed**
    - **Validates: Requirements 11.2**
  - [ ]* 7.3 Write property test for full-width coverage
    - **Property 24: Chart always covers the visible width**
    - **Validates: Requirements 11.3**
  - [ ]* 7.4 Write property test for chart segment culling
    - **Property 25: Off-screen chart segments are culled**
    - **Validates: Requirements 11.4**
  - [ ]* 7.5 Write property test for monotonic jaggedness
    - **Property 26: Chart jaggedness is monotonic in difficulty**
    - **Validates: Requirements 13.7**

- [ ] 8. Checkpoint - pure logic core complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Implement IO layer: storage, audio, canvas scaler
  - [ ] 9.1 Implement `Storage` with `getHighScore()` / `setHighScore(n)` over `localStorage`
    - Key `flappyKiro.highScore`; missing/invalid → 0; write only when `n` exceeds stored value; wrap reads/writes in try/catch so failures keep the value in memory with no error surfaced
    - _Requirements: 8.5, 9.1, 9.2, 9.3, 9.4 / Design: Storage behavior_
  - [ ]* 9.2 Write property test for high-score max/persist condition
    - **Property 18: High score is the max of session and stored**
    - **Validates: Requirements 8.5, 9.1**
  - [ ]* 9.3 Write property test for high-score storage round-trip
    - **Property 20: High score round-trips through storage**
    - **Validates: Requirements 9.2**
  - [ ] 9.4 Implement `AudioManager` for `jump` and `gameover`
    - `jump`: reset `currentTime=0` then `play()`; `gameover`: play on Game Over entry; queue blocked autoplay and retry on first keydown/pointerdown; catch and swallow all load/play errors with no UI change
    - _Requirements: 7.4, 7.5, 12.1, 12.2, 12.3, 12.4, 12.5 / Design: AudioManager behavior_
  - [ ] 9.5 Implement `CanvasScaler` that fits the canvas to the viewport preserving 800:600
    - Compute display rect preserving the 4:3 aspect ratio with letterbox/pillarbox; apply as CSS size while keeping the 800×600 backing store
    - _Requirements: 13.5 / Design: CanvasScaler_
  - [ ]* 9.6 Write property test for aspect-preserving canvas scaling
    - **Property 27: Canvas scaling preserves aspect ratio and fits the viewport**
    - **Validates: Requirements 13.5**

- [ ] 10. Implement input handler with FSM routing
  - [ ] 10.1 Implement `InputHandler` binding keydown(Space) and pointerdown, routing one input to exactly one action per FSM state
    - `preventDefault()` on Spacebar; drop `event.repeat`; in `START` start the run (no jump); in `GAME_OVER` restart after lockout (no jump) and ignore during lockout; in `PLAYING` apply a jump impulse
    - _Requirements: 2.4, 3.3, 3.4, 8.2, 8.7 / Design: InputHandler behavior_

- [ ] 11. Implement game loop, FSM, and restart (wiring)
  - [ ] 11.1 Implement the delta-time game loop over `requestAnimationFrame` with the clamp
    - `dt = min((now-last)/1000, maxDeltaTime)`; dispatch `update(dt,state)` (only in PLAYING) then `render(ctx,state)` by current screen; store `lastTime`
    - _Requirements: 14.1, 14.2, 14.3 / Design: Game Loop_
  - [ ]* 11.2 Write property test for delta-time partition independence
    - **Property 28: Delta-time partition independence**
    - **Validates: Requirements 14.1, 14.2**
  - [ ]* 11.3 Write property test for delta-time clamp
    - **Property 29: Delta-time clamp**
    - **Validates: Requirements 14.3**
  - [ ] 11.4 Implement the `START` / `PLAYING` / `GAME_OVER` FSM and the per-frame PLAYING update
    - Wire physics, obstacle/coin spawn+update, scoring, difficulty, and chart into the update step; on any collision or boundary cross transition to GAME_OVER in the same frame; on GAME_OVER entry persist high score and play game-over audio; ignore further hazard events while GAME_OVER
    - _Requirements: 2.1, 2.5, 4.1, 5.4, 7.1, 7.2, 7.3, 7.4, 7.6, 10.4_
  - [ ]* 11.5 Write property test for Game Over absorbing hazard events
    - **Property 16: Game Over absorbs further hazard events**
    - **Validates: Requirements 7.6**
  - [ ] 11.6 Implement restart with the 500 ms lockout producing a canonical fresh session
    - Record `gameOverAt`; ignore restart input for `restartLockoutMs`; a valid restart resets profit to 0, clears obstacles, restores Ghosty position/velocity, resets speed to `initialSpeed` and level to 0, and enters `PLAYING` directly
    - _Requirements: 8.2, 8.7, 10.4_
  - [ ]* 11.7 Write property test for restart lockout threshold
    - **Property 17: Restart input lockout threshold**
    - **Validates: Requirements 8.2**
  - [ ]* 11.8 Write property test for canonical fresh session
    - **Property 19: Restart produces a canonical fresh session**
    - **Validates: Requirements 8.7, 10.4**

- [ ] 12. Checkpoint - playable loop complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Implement renderer: screens, sprites, and HUD
  - [ ] 13.1 Implement the `Renderer` for the chart background and Ghosty sprite
    - Draw the procedural chart (dark bg, green up-moves, red down-moves) using Canvas 2D only; draw `assets/ghosty.png` scaled to `spriteWidth × spriteHeight`, guarding with a placeholder rectangle until the image `onload` fires
    - _Requirements: 3.1, 11.1, 11.5 / Design: Renderer, Error Handling_
  - [ ] 13.2 Render candlestick obstacles and gold coins
    - Each pillar: full-width body plus a thin centered wick toward the gap; top pillar red, bottom pillar green; draw uncollected coins at the gap midpoint in `palette.gold`
    - _Requirements: 4.2, 6.2_
  - [ ] 13.3 Render the Start Screen and the Game Over Screen
    - Start: title, Ghosty sprite, prompt referencing both Spacebar and click/tap. Game Over: "Portfolio wiped out", final Profit (non-negative integer), High Score, and a Restart button — all using the shared palette/typography
    - _Requirements: 2.1, 2.2, 2.3, 8.1, 8.3, 8.4, 8.6_
  - [ ] 13.4 Render the HUD bar with Score, High, and Volatility
    - Bottom-of-canvas centered HUD bar showing "Score: N | High: N" and a "Volatility: N" indicator equal to the current difficulty level, updated when the level changes
    - _Requirements: 5.2, 5.3, 13.3, 13.6_

- [ ] 14. Apply trading/finance visual polish
  - [ ] 14.1 Enforce the palette and typography consistently across all three screens
    - All colors drawn from `Config.palette` (dark bg, green up, red down, light text); all text ≥ `minFontSize` (14px); no screen-exclusive colors or fonts
    - _Requirements: 2.3, 13.1, 13.2_

- [ ] 15. Final integration and smoke verification
  - [ ] 15.1 Confirm `index.html` is fully self-contained and the game runs end to end
    - Verify no external URLs (inline/local references only), `Config` exposes every required constant, and a full run (start → play → score/coins → difficulty scaling → game over → restart) works under `file://`
    - _Requirements: 1.1, 1.2, 1.3_
  - [ ]* 15.2 Add optional smoke/integration/visual tests
    - Smoke: no external URLs and Config type checks; visual: palette-only colors, text ≥ 14px, green-up/red-down candlesticks and chart
    - _Requirements: 1.1, 1.3, 2.3, 4.2, 11.5, 13.1, 13.2_

- [ ] 16. Final checkpoint - game complete
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped — they are all tests (property-based, unit/example, integration, smoke) and never block or gate any game-building task.
- The shipped artifact is a single self-contained `index.html`; the optional property tests live in a dev-only Node + fast-check harness that imports the pure core functions.
- Each task references specific requirements and/or design properties for traceability.
- Checkpoints provide incremental validation points; the pure logic core is built before the IO layer, which is built before wiring, rendering, and polish.
- Property tests validate the universal correctness properties (Properties 1–30); each is its own sub-task placed near the implementation it validates.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["2.1", "2.4", "3.1", "3.2", "4.1", "5.1", "6.1", "7.1", "9.1", "9.4", "9.5"] },
    { "id": 3, "tasks": ["2.2", "2.3", "2.5", "2.6", "3.3", "3.4", "3.5", "3.6", "3.7", "4.2", "4.3", "4.4", "4.5", "5.2", "5.3", "5.4", "6.2", "6.3", "7.2", "7.3", "7.4", "7.5", "9.2", "9.3", "9.6", "10.1"] },
    { "id": 4, "tasks": ["5.5", "11.1"] },
    { "id": 5, "tasks": ["11.2", "11.3", "11.4"] },
    { "id": 6, "tasks": ["11.6"] },
    { "id": 7, "tasks": ["11.5", "11.7", "11.8", "13.1"] },
    { "id": 8, "tasks": ["13.2", "13.3", "13.4"] },
    { "id": 9, "tasks": ["14.1"] },
    { "id": 10, "tasks": ["15.1"] },
    { "id": 11, "tasks": ["15.2"] }
  ]
}
```
