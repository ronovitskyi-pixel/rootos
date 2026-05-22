# Application Specifications

Detailed specifications for built-in applications.

## Type - Text Editor

### Features
- Open and create text files
- Basic editing (insert, delete, backspace)
- Copy, cut, paste
- Find and replace
- Line numbers
- Syntax highlighting (optional)
- Auto-indent
- File save (Ctrl+S)
- File open (Ctrl+O)
- New file (Ctrl+N)

### Data Structure
```c
struct Editor {
    char *buffer;                  // Text buffer
    uint32_t buffer_size;          // Size of buffer
    uint32_t cursor_pos;           // Current cursor position
    uint32_t selection_start;      // Selection start
    uint32_t selection_end;        // Selection end
    bool modified;                 // Has unsaved changes
    char filename[256];            // Current file
    struct Window *window;
};
```

### Core Functions
- `editor_open(filename)` – Load file
- `editor_save()` – Save to disk
- `editor_insert_char(ch)` – Insert character at cursor
- `editor_delete_char()` – Delete character at cursor
- `editor_copy()` – Copy selection
- `editor_paste()` – Paste from clipboard
- `editor_find(text)` – Find text
- `editor_replace(find, replace)` – Replace text
- `editor_render()` – Draw editor to window

## Draw - Painting Application

### Features
- Free-hand drawing with brush
- Color picker
- Brush size adjustment
- Line tool (hold shift)
- Rectangle/circle tools
- Eraser
- Undo/Redo (20-level history)
- Save as PNG/BMP
- Clear canvas

### Data Structure
```c
struct Canvas {
    void *pixels;                  // Pixel buffer
    uint32_t width, height;        // Canvas size
    struct UndoEntry *undo_stack[20];  // Undo history
    uint32_t undo_depth;
    struct {
        Color color;
        int size;
        int opacity;
    } brush;
};
```

### Core Functions
- `canvas_create(width, height)` – Create canvas
- `canvas_draw_pixel(x, y, color)` – Draw pixel
- `canvas_draw_line(x0, y0, x1, y1)` – Draw line
- `canvas_undo()` – Undo last action
- `canvas_redo()` – Redo last undo
- `canvas_save(filename)` – Save image
- `canvas_clear()` – Clear canvas

## Pong Game

### Features
- Two paddles (player vs AI)
- Ball physics (bounce, spin)
- Score tracking
- Sound effects (optional)
- Pause/Resume
- Difficulty levels

### Game State
```c
struct PongGame {
    struct {
        int x, y;                  // Ball position
        int vx, vy;                // Ball velocity
        int radius;
    } ball;
    
    struct {
        int y;                     // Left paddle Y
        int score;
    } player;
    
    struct {
        int y;                     // Right paddle Y
        int score;
    } ai;
    
    bool paused;
    uint32_t difficulty;
};
```

### Game Loop
1. Handle input (player paddle)
2. AI logic (move paddle)
3. Update ball physics
4. Check collisions
5. Render game state
6. Check win condition

## Snake Game

### Features
- Snake grows when eating food
- Collision detection (walls, self)
- Score tracking
- Multiple speed levels
- High score persistence

### Game State
```c
struct SnakeGame {
    struct {
        int x, y;                  // Segment position
    } segments[256];
    uint32_t length;               // Snake length
    int vx, vy;                    // Velocity (direction)
    
    struct {
        int x, y;                  // Food position
    } food;
    
    uint32_t score;
    uint32_t speed;
    bool game_over;
};
```

## Breakout Game

### Features
- Brick grid (10x10)
- Paddle control
- Ball physics
- Power-ups (wider paddle, faster ball, multi-ball)
- Level progression
- Lives system (3 lives)

### Game State
```c
struct BreakoutGame {
    struct {
        int x, y;                  // Paddle position
    } paddle;
    
    struct {
        int x, y;
        int vx, vy;
    } balls[3];                    // Multiple balls
    uint32_t active_balls;
    
    struct {
        bool active;               // Brick exists
        uint8_t hits;              // Hits remaining
    } bricks[10][10];
    
    uint32_t lives;
    uint32_t score;
    uint32_t level;
};
```

## Settings Application

### Tabs

**Display Settings:**
- Resolution (640x480, 800x600, 1024x768, 1280x1024)
- Color depth (16, 32 bit)
- Refresh rate

**Input Settings:**
- Mouse sensitivity
- Keyboard layout
- Key repeat rate

**Theme Settings:**
- Color scheme selection
- Font size
- Window decoration style

**Network Settings:**
- DHCP enable/disable
- Static IP configuration
- DNS servers
- Proxy settings

### Data Structure
```c
struct SystemSettings {
    struct {
        uint16_t width, height;
        uint8_t color_depth;
    } display;
    
    struct {
        uint8_t sensitivity;
        char keyboard_layout[32];
    } input;
    
    struct {
        char theme_name[32];
        uint8_t font_size;
    } ui;
    
    struct {
        bool dhcp_enabled;
        uint32_t ip_address;
        uint32_t netmask;
        uint32_t gateway;
        uint32_t dns_servers[2];
    } network;
};
```

---

**Related Documentation:**
- [ARCHITECTURE.md](../ARCHITECTURE.md)
- [docs/UI_ARCHITECTURE.md](UI_ARCHITECTURE.md)