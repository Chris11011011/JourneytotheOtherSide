# Journey to the Other Side
## Crossy Road–Style FPGA Game (DE1-SoC)

A hardware implementation of a Crossy Road style game built in Verilog for the DE1-SoC FPGA board.  
The design runs entirely on an FPGA ecosystem: keyboard input, game logic, video output system, and VGA output.

By: David Jung & Christopher Lee

---

## Game Demo:
YouTube Link: https://youtu.be/LGBj9afiXnI

---

## Repository Structure

This repository is organized around these logical blocks:

- `docs/` – Diagrams, notes, and design documentation.
- `assets/` – Game art pixel assets (as .png)

> **Academic Integrity & Licensing**  
> To comply with academic integrity and plagiarismm policies at the University of Toronto, the Verilog source code for this course project will **not** be published in this repository.

---

## Game Logic Overview

The player controls a chicken attempting to cross lanes of traffic:

- **Input**: PS/2 keyboard (w, a, s, d)
- **Output**: 320x240 VGA, 9 bit colour (3:3:3)
- **FPGA Board**: DE1-SoC (Intel Cyclone V)

Every frame, the hardware:
1. Reads player input.
2. Updates game state (position, score, collisions).
3. Sequences a set of sprite engines to draw backgrond strips, cars, the chicken, and UI overlays directly to the VGA adapter.

---

## High Level Architecture

Conceptually, the design is split into three main subsystems:

1. **Input & Game Logic**
2. **Sprite Drawer**
3. **Video Arbiter**
4. **Pixel Multiplexer**

All bloks are fully synchronous to the 50 MHz system clock and coordinated by a small set of global control signals (`reset`, `tick`, `scroll`, `alive`, `game_start`).

---

### 1. Input & Game Logic

**PS/2 Input Decoder**

- Interfaces with the PS/2 keyboard.
- Decodes scan codes for the four movement keys.
- Handles key press/release using the `F0` break code.
- Outputs four clean, single cycle booleans: `up`, `down`, `left`, `right`.

**Direction encoding**

- The four directional booleans are encoded into a 2 bit direction value:
  - `00` = up, `01` = right, `10` = down, `11` = left.
- This code is used both for movment and for selecting which chicken sprite to draw (facing direction).

**Timing Generators**

- **Game tick**: divides the 50 MHz clock down to a “frame” tick used to pace horizontal motion (car movement, chicken steps).
- **Scroll tick**: a slower divider that scrolls the entire world downward in fixed 20-pixel steps.

**Chicken Logic**

- Maintains `(x, y)` tile aligned coordinates for the chicken:
- Enforces world boundaries (320 x 240).
- Increments score when the player moves upwards.
- Tracks `game_start` (has the player moved yet??) and `alive_boundary` (did they fall off the world?).

**Car Logic & World Layout**

- Cars are organized as **12 lanes**, each defined by:
  - A starting X coordinate.
  - A starting Y coordinate (linked to a particular background strip).
- Each lane has a horzontal motion engine:
  - On each game tick, car X advances by 1 pixel and wraps at the screen edge.
- A dedicated collision detector per lane checks:
  - overlap between the chicken’s 20x20 sprite and each car’s 40x20 sprite.

**Vertical Looping (World Scroll)**

- A generic “looping” module is used for:
  - Car Y positions.
  - Background strip Y positions.
- On each scroll tick:
  - Y is incrmented by 20 pixels.
  - If Y reaches the bottom (Y = 220), it wraps back to 0.
- When the chiken nears the top of the screen, background strips and cars are shifted to simulate continuous upward progress.

---

### 2. Reusable Sprite Drawer

A parallel customizable sprite drawer used for:

- 320x20 scrolling background strips (road/grass/dirt).
- 40x20 car sprites.
- 20x20 chicken sprites (one per facing direction).
- Full-screen start and game-over overlays.

**Key Responsibilities**

- Read pixel data from a ROM initialized from a `.mif` image.
- Iterate over all `(row, column)` positions inside the sprite.
- Offset each pixel by a provided on-screen `(x, y)` coordinate.
- Output `(x, y, colour, write_enable)` to the VGA adapter.

**Customizable Parameters**

Each instance can be configured by parameters:

- `SPR_W`, `SPR_H` – sprite width & height.
- `COLOR_DEPTH` – bits per pixel.
- `SPRITE_IMAGE` – path to to the MIF file.
- `SKIP_ZERO` – optional "black = transparent" behavior.

**Internal FSM**

1. **IDLE**
   - Wait for a one cycle `start` pulse.
   - Reset internal X/Y counters and ROM address.

2. **PLACE_ADDRESS**
   - Compute on-screen pixel position:
     - `screen_x = sprite_x_offset + inner_x`
     - `screen_y = sprite_y_offset + inner_y`
   - Place the ROM address for the current pixel.

3. **WAIT_FOR_ROM**
   - Wait one clock for ROM data to appear.
   - Latch the colour.

4. **WRITE_PIXEL**
   - Perform checks:
     - Bounds: only draw if within 320x240.
     - Transparency: optionally skip pure black.
   - Assert `write_enable` for one cycle if valid.
   - Advance X/Y counters and ROM addresss.
   - When the last pixel is processed, raise `done` and return to **IDLE**.

---

### 3. Video Arbiter

To manage many sprite drawers without a full frame buffer, the design uses a **central arbiter FSM** that controls when each sprite drawer runs.

1. `IDLE`  
   - Wait for the globel `tick` from the game logic.

2. **Background phase**
   - Sequentially start background strip drawers:
     - `BG0 → BG1 → ... → BG11`
   - Each step:
     - Assert the start pulse for that strip’s drawer.
     - Wait for its `done` signal.
     - Move to the next background strip.

3. **Foreground car phase**
   - Sequentially draw car sprites:
     - `SPRITE0 → SPRITE1 → ... → SPRITE11`.

4. **Chicken phase**
   - Based on the frozen direction code, choose one of the four chicken sprite drawers (up/right/down/left).
   - Start that drawer; wait for it's done`.

5. **Overlay phase**
   - If `alive == 0`: draw the **gameover** screen.
   - Else if `game_start == 0`: draw the **start** screen.
   - Else: no overlay; go back to `IDLE`.

This defines a strict **layer order**:

> Background strips → Cars → Chicken → Overlays and guarantees that, on each tick, the entire scene is redrawn in sequence.

---

### 4. Pixel Multiplexer & VGA Output

Because many sprite drawers exist in parallel, but only one should drive the VGA adapter at a time, `top_video` contains a **pixel multiplexer**.

- Inputs:
  - `(x, y, colour, write)` from each background strip, each car, the chicken (directional), start screen, and game-over screen.
- Selector:
  - The current state of the arbiter FSM.
- Output:
  - A single stream of pixels forwarded to the `vga_adapter`.

For example:

- In `DRAW_BG3`, the MUX forwards pixels from background strip 3.
- In `DRAW_SPRITE5`, it forwards pixels from car sprite 5.
- In `DRAW_CHICKEN`, it forwards pixels from the selected chicken drawer.
- In `DRAW_GAMEOVER_SCREEN`, it forwards the overlay pixels.

The VGA adapter then takes:

- 9 bit colour,
- 9 bit X coordinate,
- 8 bit Y coordinate,
- `write` strobe,

and generates the analog VGA signals for the display.

---

## Gameplay & Controls

- **Reset**: hardware reset button (active-low).
- **Movement**: movement keys via PS/2 keyboard.
- **Score**: incremented on each upwardd move.
- **Game start**: first valid move transitions from the start screen to gameplay.
- **Game over**: triggered on colision or falling off screen; gameover screen is drawn until reset.

---

## Future Improvements

Several enhancements are possible without fundamentally changing the architecture:

1. **Double Buffering for Flicker Reduction**
   - Introduce two frame buffers.
   - Draw into one while the VGA scans out of the other.
   - Swap buffers once a frame is complete.
   - This would largely eliminate visible flicker and tearing, at the expense of memory and controller complexity.

2. **Audio Effects**
   - Add simple audio output driven by game events:
     - Hops, colisions, and gameover tones.
   - Could be implemented using an audio codec.

3. **Second Player**
   - Add another character with separate controls and a sprite set.
   - Duplicate colision logic for the second player.
   - Extend the arbiter and MUX to render another sprite layer.

4. **Static Obstacles & Environment Variety**
   - Introduce additional background or foreground sprites (e.g., logs, barriers).
   - Extend colision rules to differentiate between "hazard", "blocking", and “safe” tiles.

---
