# CLAUDE.md - AI Assistant Guidelines for 2D Subway Surf

## Project Overview

This is a **2D Subway Surfers-style game** that runs on bare-metal ARM A9 hardware (DE1-SoC FPGA Development Board). The player controls a character avoiding incoming trains across three parallel tracks. The game is written entirely in C with no operating system, using direct memory-mapped I/O for graphics and input.

**Key Characteristics:**
- Real-time embedded game (no OS)
- Resolution: 320x240 pixels, 16-bit RGB565 color
- Double-buffered rendering with V-sync
- Interrupt-driven input via pushbuttons
- Progressive difficulty (speed increases over time)

## Codebase Structure

```
2D_Subway_Surf/
├── test.c                    # Main game code (2,238 lines) - ALL game logic
├── README.md                 # Brief project description
├── Attribution Report.pdf    # Attribution documentation
├── CLAUDE.md                 # This file
├── .vscode/                  # VSCode configuration
│   ├── launch.json           # GDB debugger config
│   ├── settings.json         # C/C++ runner settings
│   └── c_cpp_properties.json # Intellisense config
└── sourse_file/              # Supporting headers (note: typo in name)
    ├── address_map.h         # FPGA memory-mapped I/O addresses
    ├── game_objects.h        # Game struct definitions
    └── display.h             # Empty placeholder
```

## Key File: test.c Structure

The entire game is in `test.c`, organized into these sections:

| Lines | Section | Description |
|-------|---------|-------------|
| 1-72 | Includes & Globals | Headers, constants, global variables |
| 73-101 | Structures | Player, Train, Coin struct definitions |
| 105-220 | Graphics Functions | draw_pixel, draw_line, draw_rectangle, wait_for_vsync |
| 222-338 | Game Rendering | draw_score, draw_background, draw_player, draw_train |
| 344-397 | Initialization | initialize_player, initialize_obstacles, initialize_game |
| 399-556 | Interrupts | ARM A9 interrupt handling, pushbutton ISR |
| 584-991 | Game Logic | spawn_trains, update_game, collision detection |
| 993-2113 | Sprite Data | Inline pixel arrays for player, trains, background |
| 2114-2237 | Main Loop | Entry point and game loop |

## Development Conventions

### Naming Conventions

- **Functions:** `snake_case` (e.g., `draw_pixel()`, `initialize_game()`)
- **Macros/Constants:** `UPPER_CASE` (e.g., `SCREEN_WIDTH`, `LEFT_TRACK`)
- **Variables:** `snake_case` with descriptive names (e.g., `pixel_buffer_start`)
- **Struct Types:** `PascalCase` (e.g., `Player`, `Train`, `Coin`)

### Hardware Access Pattern

```c
// Write to hardware register
*(volatile int *)ADDRESS = value;

// Read from hardware register
int value = *(volatile int *)ADDRESS;
```

Always use `volatile` keyword when accessing memory-mapped I/O.

### Interrupt-Related Variables

Variables shared with interrupt handlers must be declared `volatile`:
```c
volatile int tick = 0;
volatile int key_dir = 0;
volatile int pixel_buffer_start;
```

### Sprite Data Format

Sprites are stored as inline arrays of 16-bit RGB565 color values:
```c
short sprite_name[] = {
    0x0000, 0xFFFF, 0xF800, ...  // Each value is one pixel
};
```

### Graphics Coordinates

- Origin (0,0) is top-left corner
- X increases rightward (0-319)
- Y increases downward (0-239)

## Key Constants

```c
#define SCREEN_WIDTH    320
#define SCREEN_HEIGHT   240

// Track X positions (center of each lane)
#define LEFT_TRACK      109
#define MIDDLE_TRACK    145
#define RIGHT_TRACK     181

// Train sizes
#define SHORT_TRAIN     80   // pixels height
#define LONG_TRAIN      120  // pixels height
```

## Critical Hardware Addresses

From `sourse_file/address_map.h`:

| Address | Name | Purpose |
|---------|------|---------|
| 0xFF203020 | PIXEL_BUF_CTRL_BASE | Video output control |
| 0xC9000000 | FPGA_CHAR_BASE | Text/character buffer |
| 0xFF200050 | KEY_BASE | Pushbutton input |
| 0xFFFED000 | MPCORE_GIC_DIST | Interrupt distributor |
| 0xFFFEC100 | MPCORE_GIC_CPUIF | Interrupt CPU interface |

## Game Mechanics

### Input Mapping
- **KEY0** (key_dir = 1): Move right
- **KEY2** (key_dir = 2): Move left
- **KEY3** (key_dir = 3): Reset/respawn after game over

### Game Loop Flow
1. Clear screen
2. Process input (from interrupt-set flags)
3. Update game state (player movement, train spawning, difficulty)
4. Collision detection
5. Render (background → trains → player → score)
6. V-sync and buffer swap

### Collision Detection
Uses AABB (Axis-Aligned Bounding Box) intersection via `rectanglesCollide()`.

## Build & Debug

**Compiler:** GCC (MinGW64 on Windows cross-compile)

**Compiler Flags:**
```
-Wall -Wextra -Wpedantic -Wshadow -Wformat=2 -Wconversion
```

**Debug:** GDB through VSCode launch configuration

## Guidelines for AI Assistants

### When Modifying Code

1. **Preserve volatile qualifiers** - Critical for hardware access and interrupt safety
2. **Maintain interrupt handler attributes** - Functions like `__cs3_isr_irq` use `__attribute__((interrupt))`
3. **Keep memory addresses exact** - Never change hardware address constants
4. **Test collision bounds** - Player movement and train positions must stay within track boundaries

### Code Style Requirements

1. Use `snake_case` for new functions
2. Add new sprite data to the sprite data section (lines 993-2113)
3. Keep game logic in the update functions section
4. Use `volatile` for any new interrupt-shared variables

### Performance Considerations

1. This is a real-time system - avoid expensive operations in the game loop
2. Graphics operations directly write to memory - no buffering
3. Double-buffering is used - always work with `pixel_buffer_start` variable
4. V-sync wait is blocking - essential for smooth rendering

### Common Modifications

**Adding a new sprite:**
1. Create a `short sprite_name[]` array with RGB565 pixel values
2. Add a `draw_sprite_name()` function following existing patterns
3. Call draw function in `render_game()`

**Adding a new game object:**
1. Define struct in the structures section (or `game_objects.h`)
2. Add initialization function
3. Add update logic in `update_game()`
4. Add rendering in `render_game()`

**Adding new input:**
1. Modify `pushbutton_ISR()` to handle new KEY values
2. Process new `key_dir` values in main game loop

### Known Issues to Be Aware Of

1. **Typo in directory name:** `sourse_file` should be `source_file`
2. **display.h is empty** - Placeholder file
3. **Magic numbers** - Many hard-coded values throughout
4. **No modular structure** - All code in single file
5. **Limited error handling** - Bounds checking is minimal

### Testing

No automated test framework. Testing requires:
1. DE1-SoC FPGA hardware, OR
2. Hardware emulator/simulator

Manual testing should verify:
- Player moves correctly between tracks
- Trains spawn and move at correct speeds
- Collision detection works at boundaries
- Score increments properly
- Game over screen appears on collision
- Reset (KEY3) works correctly

## Quick Reference

**Entry Point:** `main()` at line 2114

**Key Functions:**
- `draw_pixel(x, y, color)` - Draw single pixel
- `draw_rectangle(x1, y1, x2, y2, color)` - Draw filled rectangle
- `update_game(...)` - Main game state update
- `render_game(...)` - Render current frame
- `rectanglesCollide(...)` - Collision detection
- `spawn_trains(...)` - Dynamic train spawning
