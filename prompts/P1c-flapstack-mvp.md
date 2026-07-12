# P1c: Flapstack MVP (supersedes P1 and P1b)

## Objective
A playable single-file HTML prototype of Flapstack: a four-block piece with flappy-bird physics auto-scrolls toward a stacking well and must be landed into the stack. One file, open in browser, play immediately. This supersedes P1 and P1b entirely; if either was partially executed, discard and start from this spec.

## Current state
Greenfield. Create a new repo folder `flapstack` with a single `index.html`. Nothing exists yet.

## Constraints
- One file: `index.html` with inline CSS and JS. Canvas 2D. Zero dependencies, zero build step. This deviates from the usual TypeScript stack deliberately: it is a physics-feel prototype and iteration speed beats tooling.
- Fixed logical resolution 960x540, scaled to fit window with letterboxing. Scale the canvas backing store by devicePixelRatio so rendering is crisp on high-DPI displays; all game logic stays in logical pixels.
- 60fps target with a fixed-timestep update (accumulator pattern).
- Desktop input only for MVP: Space or click = flap, R or ArrowUp = rotate 90 degrees clockwise. Mobile is out of scope.
- Branding: the game is called Flapstack. Title on the start overlay, page <title>, and game-over screen. Never use the word Tetris or any variant anywhere in code, comments, UI text, file names, or the repo name. Refer to pieces as "pieces" or "blocks" and the play area as "the well" or "the stack".
- Piece colors must NOT use the standard falling-block genre palette (cyan I, yellow O, purple T, green S, red Z, blue J, orange L). Use this palette instead, assigned as listed: I #E8604C (coral), O #2D9C8A (teal), T #E8B84C (amber), S #7B6CB2 (violet), Z #4C86E8 (azure), J #C24C8A (magenta), L #8AA84C (olive). Background dark slate #1E2430, well border and HUD text off-white #EDE8DC.
- No em dashes in any code comments or UI text.

## Game design spec

### Layout
Flight zone occupies the left ~600px. The well occupies the right side: 10 columns x 16 rows, cell size 30px, well is 300px wide x 480px tall, vertically centered, right-aligned with a 30px margin. Well has a visible border on its top, right, and bottom edges (left edge open for entry) and faint grid lines.

### Piece lifecycle
1. A four-block piece (7-bag randomizer over the standard 7 shape geometries) spawns at x = 40, vertically centered in the flight zone, rendered at grid cell scale.
2. It auto-scrolls right at constant speed (start 120 px/s). Gravity pulls it down (900 px/s^2), each flap applies an instant upward velocity set (not additive), vy = -320. Terminal fall speed capped at 600 px/s. Tune these constants so the feel is close to Flappy Bird; expose all of them in a single CONFIG object at the top of the script.
3. Rotation input rotates the piece 90 degrees CW instantly around its bounding-box center, mid-flight, unlimited uses. If the rotated position would intersect the obstacle, the stack, the well walls, or leave the screen, the rotation is a no-op (piece stays in its current orientation, no penalty). The piece moves in continuous coordinates during flight; no grid snapping until lock.
4. Exactly one pipe-style obstacle stands in the flight zone per flight: a vertical pair (top and bottom bar, 60px wide) with a gap of 180px at a random vertical position (clamped so the gap is fully on screen), located at x = 300. Passing through it scores 10 points, awarded at most once per flight.

### Lock rule (directional)
A lock is triggered only by downward contact: any block's bottom edge meets the well floor or the top face of an occupied cell, while the piece's full horizontal extent is within the well's column range. On lock, snap the piece origin (its reference corner in continuous coordinates) to the nearest grid cell and derive all four block positions from that origin; never round blocks independently, which can distort the shape. If any derived cell overlaps an occupied cell or lies outside the well, the landing is invalid and the run ends. If valid, write cells into the grid, check and clear full rows (100 points per line, standard shift-down), then spawn the next piece on the left. By construction a locked piece always has at least one block supported directly beneath it; floating locks must be impossible.

### Death conditions (all active)
- Mid-flight collision with the obstacle.
- Contact with the flight-zone ceiling (y = 0) or floor (y = 540) anywhere left of the well columns.
- Contact with the top band above the well or the bottom band below it (the 30px strips within the well's x-range but outside its rows).
- Horizontal contact with a side face of the stack or with the well's right wall.
- Reaching the right screen edge without locking.
- Invalid landing snap (overlap or out-of-well cells).
- Top-out: after a valid lock, any occupied cell in the top 2 rows of the well.

Collision detail: use the piece's actual 4 blocks (AABB per block), not its bounding box, for all collision checks.

### HUD and flow
Current score top-left, lines cleared, and a small preview of the next piece. Start overlay before the first run showing the Flapstack title and the two controls in one line. Game-over overlay shows final score and "press Space to restart". On entering GAMEOVER, ignore all input for 500ms so a panic flap does not skip the score screen; after the lockout, Space restarts with an empty well, score 0, and a fresh 7-bag.

### Difficulty ramp
Scroll speed increases 5 px/s per locked piece, capped at 220 px/s.

## Implementation
1. Scaffold `index.html`: DPR-aware canvas, CONFIG object (physics constants and palette), fixed-timestep game loop, state machine (START, FLYING, GAMEOVER).
2. Piece data: the 7 shapes as cell-offset arrays with 4 rotation states each, precomputed. Track the piece by an origin point plus rotation index; blocks derive from it in both flight and lock.
3. Flight physics, input handling, rotation with intersection check and no-op fallback.
4. Obstacle generation, pass detection, collision.
5. Well grid, directional lock detection, origin snap, lock validation, line clear, top-out check.
6. HUD, overlays, restart flow with input lockout, 7-bag, next-piece preview, difficulty ramp.

## Acceptance criteria
- Opening `index.html` directly (file://) shows the start overlay with the Flapstack title; Space begins a run.
- Flapping visibly counteracts gravity; the piece falls when idle and dies on flight-zone floor or ceiling contact.
- Flying into the obstacle ends the run; flying through the gap adds exactly 10 points once.
- R rotates the piece mid-flight through all 4 states for every shape; a rotation that would intersect the stack or obstacle does nothing and the game continues.
- A piece descending onto the stack top or well floor locks with all blocks on integer grid coordinates and the shape unchanged (no distortion, no half-cell offsets).
- A piece flying horizontally into the side of a tall stack column ends the run rather than locking midair; no locked piece ever lacks support beneath at least one block.
- Flying over the well's top edge or reaching the right wall or right screen edge ends the run.
- Filling a complete row clears it and rows above shift down; score increases by 100 per line.
- After a lock leaves blocks in the top 2 well rows, the run ends with the game-over overlay.
- Input within 500ms of game over is ignored; Space after the lockout restarts with an empty well and score 0.
- Rendered piece colors match the specified palette hexes; grep of the repo for the string "tetris" (case-insensitive) returns zero hits.
- Canvas is sharp on a high-DPI display (no visible blur at DPR 2).
- No console errors during a 2-minute play session.

## Out of scope
Mobile/touch input, portrait layout, sound, hold piece, soft/hard drop, wall kicks, scoring combos, persistence/high scores, multiple obstacles per flight, art assets beyond flat-color rectangles. Portrait and touch are P2 after desktop physics feel is validated.

## QA / verification
Playwright in headed mode on the M4 Pro (headless has been flaky; do not report headless passes). Expose `window.__game` with `state`, `grid`, `score`, and a debug helper `debugFillRow(row, skipCol)`. Automate: load page, assert start overlay, press Space, assert state FLYING, simulate flaps, assert no console errors over 30 simulated seconds; assert all locked cells have integer grid coordinates and each lock event has at least one supported block; inject a near-full row via `debugFillRow` and assert the clear adds 100 points; assert the 500ms game-over input lockout by sending Space at 100ms and 700ms after death and checking state. Physics feel is verified manually by Rabi.
