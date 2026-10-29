# Requirements Document

## Introduction

Flappy Kiro is a retro endless-scroller browser game with a trading/finance theme. The player controls Ghosty, a ghost sprite, navigating through candlestick-style obstacles on a live-scrolling stock chart background. The game runs entirely in the browser from a single `index.html` file — no backend, no frameworks, no build tools. Difficulty increases over time ("market volatility"), and a high score is persisted via localStorage.

## Glossary

- **Game**: The Flappy Kiro application running in the browser.
- **Ghosty**: The player-controlled ghost character rendered from `assets/ghosty.png`.
- **Obstacle**: A candlestick-style pillar pair (one from the top, one from the bottom) with a gap through which Ghosty must fly.
- **Gap**: The vertical opening between an obstacle pair that Ghosty passes through.
- **Coin**: A gold collectible that may appear inside a gap.
- **Profit**: The in-game score unit, incremented by passing through gaps or collecting coins.
- **Market Volatility**: The game's difficulty level, which increases as Profit grows.
- **Portfolio**: The player's accumulated score, used in "Portfolio wiped out" game-over messaging.
- **Config**: The central JavaScript configuration object holding all tunable game constants.
- **Canvas**: The HTML `<canvas>` element on which the game is rendered.
- **High Score**: The highest Profit value achieved, persisted in `localStorage`.
- **Start Screen**: The initial screen displayed before gameplay begins.
- **Game Over Screen**: The screen displayed when the player's run ends.
- **Difficulty Level**: The current difficulty tier, computed as `floor(Profit / volatilityStep)`.

---

## Requirements

### Requirement 1: Game Bootstrap and Entry Point

**User Story:** As a player, I want to open `index.html` in a browser and immediately see the game, so that I can play without any installation or server setup.

#### Acceptance Criteria

1. THE Game SHALL run entirely from a single `index.html` file opened directly in a browser via the `file://` protocol, with all JavaScript and CSS either inline within that file or referenced as local relative file paths, and with no external URLs, no server, and no build step required.
2. WHEN a player opens `index.html` in the current stable release of Chrome, Firefox, Edge, or Safari, THE Game SHALL render a visible HTML Canvas element with no JavaScript errors logged to the browser console.
3. THE Config object SHALL define at least the following tunable constants: gravity (px/s²), jump strength/impulse (px/s), initial scroll speed (px/s), maximum scroll speed (px/s), gap size (px), gap reduction per level (px), minimum gap size (px), volatility step (score interval per level), coin bonus value, coin spawn probability, and obstacle spawn interval.

---

### Requirement 2: Start Screen

**User Story:** As a player, I want to see a welcoming start screen before the game begins, so that I know how to start playing.

#### Acceptance Criteria

1. WHEN the Game loads, THE Game SHALL display a Start Screen before any gameplay begins.
2. THE Start Screen SHALL display the game title, the Ghosty character sprite, and a visible prompt that references both the Spacebar and click/tap as valid inputs to begin.
3. THE Start Screen SHALL apply the same color palette and font styles used in the gameplay UI, with no colors or typography that appear exclusively on the Start Screen.
4. WHEN the player presses Spacebar or clicks/taps the canvas while the Start Screen is displayed, THE Game SHALL hide the Start Screen and begin the active gameplay state.
5. WHILE the Start Screen is displayed and no input has been received, THE Game SHALL keep the Start Screen visible indefinitely.

---

### Requirement 3: Player Character (Ghosty)

**User Story:** As a player, I want to control Ghosty with simple input, so that I can navigate through obstacles.

#### Acceptance Criteria

1. THE Game SHALL render Ghosty using the sprite from `assets/ghosty.png` at a horizontal position fixed at 20% of the Canvas width and an initial vertical position centered at 50% of the Canvas height.
2. THE Game SHALL apply a constant downward gravity to Ghosty's vertical velocity each frame, where the gravity magnitude is defined in the Config object in units of pixels per second squared (px/s²).
3. WHEN the player presses Spacebar during active gameplay, THE Game SHALL apply a single upward impulse to Ghosty's vertical velocity, where the impulse magnitude is defined in the Config object in units of pixels per second (px/s).
4. WHEN the player clicks or taps the Canvas during active gameplay, THE Game SHALL apply the same upward impulse to Ghosty's vertical velocity as defined in criterion 3.
5. THE Game SHALL fix Ghosty's horizontal position at 20% of the Canvas width throughout gameplay, regardless of Canvas resize or game state changes.
6. THE jump impulse magnitude, gravity value, and all related physics constants SHALL be defined in the Config object.
7. THE Game SHALL use a rectangular hitbox for Ghosty that is 80% of the sprite's rendered width and 80% of the sprite's rendered height, centered on the sprite, for all collision detection with obstacles, coins, and canvas boundaries.

> Note: If Ghosty's hitbox crosses the top boundary of the Canvas, the Game transitions to the Game Over state as defined in Requirement 7, criterion 2. There is no ceiling clamp.

---

### Requirement 4: Obstacles

**User Story:** As a player, I want candlestick-style obstacles to scroll toward me, so that the game presents a continuous challenge.

#### Acceptance Criteria

1. THE Game SHALL generate obstacle pairs — one pillar from the top of the Canvas and one from the bottom — continuously during active gameplay.
2. THE Obstacles SHALL be drawn as candlestick shapes: each pillar SHALL have a rectangular body spanning the full obstacle width, plus a thin centered wick extending beyond the body toward the Gap opening. The top pillar SHALL be red; the bottom pillar SHALL be green.
3. THE Obstacles SHALL scroll from right to left across the Canvas at the current scroll speed defined in Config.
4. WHEN the right edge of an Obstacle pair reaches x = 0 (the left edge of the Canvas), THE Game SHALL remove it from the active obstacle list.
5. THE vertical Gap between each obstacle pair SHALL be sized according to the gap size constant in Config.
6. THE Gap center position SHALL be chosen randomly such that the gap center Y coordinate is at least half the gap size away from both the top and bottom edges of the Canvas, ensuring both pillars remain fully visible within the Canvas bounds.
7. THE Game SHALL spawn new obstacle pairs at regular intervals defined in Config.

---

### Requirement 5: Scoring (Profit)

**User Story:** As a player, I want to earn profit by flying through gaps, so that I have a clear measure of progress.

#### Acceptance Criteria

1. WHEN Ghosty's horizontal position crosses the right edge of an obstacle pair's Gap without collision, THE Game SHALL increment Profit by 1, counting each obstacle pair at most once per pass.
2. WHILE a game session is active, THE Game SHALL display the current Profit value on the Canvas.
3. THE Profit display SHALL be positioned within the bottom 10% of the Canvas height, horizontally centered, as shown in the example-ui layout reference.
4. WHEN a new game session starts, THE Game SHALL reset the Profit value to 0 before the first obstacle appears.

---

### Requirement 6: Gold Coins

**User Story:** As a player, I want to collect gold coins for bonus profit, so that I have extra reward for skillful flying.

#### Acceptance Criteria

1. WHEN a Gap is created during active gameplay, THE Game SHALL spawn a gold Coin inside that Gap with a probability defined in the Config object (default: 50%).
2. THE Coin SHALL be positioned at the horizontal midpoint of the Gap's width and the vertical midpoint of the Gap's opening height.
3. WHEN Ghosty's hitbox overlaps any part of a Coin's hitbox, THE Game SHALL collect the Coin and add the coin bonus value (defined in Config) to Profit.
4. WHEN a Coin is collected, THE Game SHALL remove it from the Canvas.
5. WHEN a Gap passes off-screen without the Coin being collected, THE Game SHALL remove the Coin from the Canvas.
6. THE coin bonus value and spawn probability SHALL be defined in the Config object (defaults: +5 Profit, 50% spawn chance).

---

### Requirement 7: Collision Detection and Game Over

**User Story:** As a player, I want the game to end when Ghosty hits an obstacle, the ground, or the ceiling, so that the game has meaningful consequences for mistakes.

#### Acceptance Criteria

1. WHEN Ghosty's rectangular bounding box intersects any Obstacle pillar's rectangular bounding box, THE Game SHALL transition to the Game Over state within the same game loop frame in which the intersection is detected.
2. WHEN the top edge of Ghosty's rectangular bounding box moves above the top boundary of the Canvas, THE Game SHALL transition to the Game Over state within the same game loop frame in which the boundary crossing is detected.
3. WHEN the bottom edge of Ghosty's rectangular bounding box moves below the bottom boundary of the Canvas, THE Game SHALL transition to the Game Over state within the same game loop frame in which the boundary crossing is detected.
4. WHEN the Game transitions to the Game Over state, THE Game SHALL play the sound from `assets/game_over.wav`.
5. IF the `assets/game_over.wav` file cannot be played, THEN THE Game SHALL still complete the transition to the Game Over state without interruption.
6. WHILE the Game is already in the Game Over state, THE Game SHALL ignore any subsequent collision or boundary-crossing events.

---

### Requirement 8: Game Over Screen

**User Story:** As a player, I want to see my final score and high score after dying, so that I know how well I did and can try to beat my best.

#### Acceptance Criteria

1. WHEN the Game transitions to the Game Over state, THE Game SHALL display the Game Over Screen.
2. WHEN the Game transitions to the Game Over state, THE Game SHALL ignore all Restart inputs (Spacebar, click, or tap) for the first 500 ms after the transition.
3. THE Game Over Screen SHALL display the message "Portfolio wiped out".
4. THE Game Over Screen SHALL display the final Profit from the ended session as a non-negative integer number of coins.
5. WHEN the Game Over Screen is displayed, THE Game SHALL update the stored High Score to the current session's Profit if it exceeds the stored value, then display the resulting High Score; IF no prior High Score exists, THEN THE Game SHALL display the current session's Profit as the High Score.
6. THE Game Over Screen SHALL display a Restart button.
7. WHEN the player activates the Restart button (click, tap, or Spacebar) after the 500 ms input-ignore window has elapsed, THE Game SHALL reset the Profit to zero, remove all active obstacles from the screen, restore the player character to its starting position and state, reset the game speed to its initial value, and immediately begin a new active gameplay session.

---

### Requirement 9: High Score Persistence

**User Story:** As a player, I want my high score saved between sessions, so that my best run is remembered even after closing the browser.

#### Acceptance Criteria

1. WHEN a game session ends, IF the final Profit exceeds the value stored in `localStorage` under the high score key, THEN THE Game SHALL write the final Profit value to `localStorage`.
2. WHEN the Game loads, THE Game SHALL read the High Score from `localStorage` and display it on the Game Over Screen.
3. IF no High Score exists in `localStorage`, THEN THE Game SHALL treat the High Score as 0.
4. IF writing to `localStorage` fails, THEN THE Game SHALL retain the High Score value in memory for the duration of the session without persisting it and without presenting any error to the player.

---

### Requirement 10: Difficulty Scaling (Market Volatility)

**User Story:** As a player, I want the game to get harder as my score increases, so that skilled play is rewarded with more challenge.

#### Acceptance Criteria

1. THE Game SHALL compute the current Difficulty Level as `floor(Profit / volatilityStep)`, where `volatilityStep` is defined in the Config object.
2. WHEN the Difficulty Level increases by one or more levels (including when a coin bonus causes Profit to jump past one or more multiples of `volatilityStep`), THE Game SHALL apply the scroll speed increase and gap reduction for each level gained in sequence, clamping speed at `maxSpeed` and gap size at `minGap` defined in Config, so that no level is ever skipped.
3. THE Config object SHALL define: `volatilityStep` (score interval per level), `initialSpeed` (px/s), `speedIncrement` (px/s per level), `maxSpeed` (px/s), `initialGap` (px), `gapReduction` (px per level), and `minGap` (px).
4. THE current Difficulty Level SHALL reset to 0 only when a new game session begins.

---

### Requirement 11: Background — Live Scrolling Stock Chart

**User Story:** As a player, I want to see a scrolling stock chart as the background, so that the trading theme is visually immersive.

#### Acceptance Criteria

1. WHILE the game is in the active gameplay state, THE Game SHALL render a procedurally generated stock chart occupying the full Canvas background, drawn using the Canvas 2D API without external image assets.
2. WHILE the game is in the active gameplay state, THE Game SHALL scroll the stock chart horizontally from right to left at the same pixel-per-frame rate as the obstacle scroll speed.
3. THE Game SHALL generate a new chart segment and append it to the right edge of the chart before the current rightmost segment scrolls fully into view, ensuring no gap or repeated section is visible to the player.
4. IF a chart segment would scroll fully off the left edge of the Canvas, THEN THE Game SHALL remove that segment from the rendering pipeline.
5. THE stock chart SHALL render using a dark background and chart lines or bars drawn in green to indicate upward price movement and red to indicate downward price movement.

---

### Requirement 12: Audio

**User Story:** As a player, I want sound effects for jumping and game over, so that the game feels responsive and satisfying.

#### Acceptance Criteria

1. WHEN Ghosty performs a jump, THE Game SHALL play the audio from `assets/jump.wav`, restarting playback from the beginning if the sound is already playing.
2. WHEN the Game transitions to the Game Over state, THE Game SHALL play the audio from `assets/game_over.wav`.
3. IF the browser prevents audio playback before user interaction, THEN THE Game SHALL queue the blocked audio and attempt playback on the first keypress or pointer event after page load.
4. IF an audio playback attempt fails, THEN THE Game SHALL suppress the error such that no error message, dialog, or visible UI change is presented to the player.
5. IF an audio file fails to load, THEN THE Game SHALL continue gameplay without that sound effect and display no error indication to the player.

---

### Requirement 13: UI and Visual Design

**User Story:** As a player, I want the game to look polished with a consistent trading/finance theme, so that the experience feels cohesive.

#### Acceptance Criteria

1. THE Game SHALL use a consistent color palette inspired by trading/finance interfaces throughout all screens, anchored by: a dark background, green elements for upward/positive market movements, red elements for downward/negative market movements, and light-colored text on dark surfaces.
2. THE Game SHALL render all on-screen text at a minimum font size of 14px.
3. THE Game SHALL render the current Profit and High Score in the format "Score: N | High: N" within a visually distinct HUD bar at the bottom of the Canvas, horizontally centered.
4. THE Game SHALL maintain a rendering frame rate of at least 60 fps during active gameplay on hardware capable of rendering the Canvas content.
5. THE Game SHALL use a fixed logical canvas resolution of 800 × 600 pixels. THE Canvas SHALL be scaled to fill the browser viewport while preserving this 800:600 aspect ratio, applying letterboxing (horizontal bars) or pillarboxing (vertical bars) as needed.
6. THE HUD SHALL display a 'Volatility: N' indicator showing the current Difficulty Level (N), updated whenever the Difficulty Level changes.
7. WHEN the Difficulty Level increases, THE Game SHALL increase the amplitude or jaggedness of the procedurally generated stock chart background proportionally to the current Difficulty Level, so higher volatility levels visually reflect more erratic market movement.

---

### Requirement 14: Frame-Rate Independence

**User Story:** As a player, I want the game to run at the same speed on 60 Hz and 144 Hz displays, so that physics and scrolling feel consistent on any hardware.

#### Acceptance Criteria

1. THE Game SHALL drive all physics (gravity, jump impulse, Ghosty velocity), obstacle scroll speed, coin movement, and background chart scrolling using delta-time (time elapsed since the last frame in seconds), so that gameplay speed is independent of display refresh rate.
2. THE Config object SHALL express all velocity and acceleration values in units per second (e.g., gravity in px/s², jump impulse in px/s, scroll speed in px/s), and the game loop SHALL convert these to per-frame deltas by multiplying by the delta time each frame.
3. THE Game SHALL cap the maximum delta time applied in any single frame at 100 ms to prevent large position jumps after tab switches or rendering stalls.
