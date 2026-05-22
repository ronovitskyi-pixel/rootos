# Graphics Pipeline & Rendering Architecture

Framebuffer-based graphics system with software rendering primitives.

## Framebuffer Setup

```c
struct Framebuffer {
    void *base_address;            // Virtual address of framebuffer
    uint32_t width;                // Horizontal resolution
    uint32_t height;               // Vertical resolution
    uint32_t bytes_per_pixel;      // 1, 2, 3, or 4
    uint32_t pitch;                // Bytes per scanline
    uint32_t total_size;           // Width * Height * bytes_per_pixel
} fb;

// Initialize framebuffer from bootloader info
void init_graphics(struct BootInfo *boot_info) {
    fb.base_address = (void*)(uint64_t)boot_info->video.framebuffer_addr;
    fb.width = boot_info->video.width;
    fb.height = boot_info->video.height;
    fb.bytes_per_pixel = boot_info->video.bits_per_pixel / 8;
    fb.pitch = fb.width * fb.bytes_per_pixel;
    fb.total_size = fb.height * fb.pitch;
}
```

## Color Format (ARGB 32-bit)

```c
typedef uint32_t Color;

#define COLOR(a, r, g, b) ((a << 24) | (r << 16) | (g << 8) | b)
#define BLACK        COLOR(0xFF, 0x00, 0x00, 0x00)
#define WHITE        COLOR(0xFF, 0xFF, 0xFF, 0xFF)
#define RED          COLOR(0xFF, 0xFF, 0x00, 0x00)
#define GREEN        COLOR(0xFF, 0x00, 0xFF, 0x00)
#define BLUE         COLOR(0xFF, 0x00, 0x00, 0xFF)
#define GRAY         COLOR(0xFF, 0x80, 0x80, 0x80)
#define DARK_GRAY    COLOR(0xFF, 0x40, 0x40, 0x40)
```

## Basic Rendering Operations

```c
// Set single pixel
void put_pixel(int x, int y, Color color) {
    if (x < 0 || x >= fb.width || y < 0 || y >= fb.height) return;
    
    uint32_t offset = y * fb.pitch + x * fb.bytes_per_pixel;
    *(Color*)((uint8_t*)fb.base_address + offset) = color;
}

// Get pixel color
Color get_pixel(int x, int y) {
    if (x < 0 || x >= fb.width || y < 0 || y >= fb.height) return 0;
    
    uint32_t offset = y * fb.pitch + x * fb.bytes_per_pixel;
    return *(Color*)((uint8_t*)fb.base_address + offset);
}

// Fill rectangle
void fill_rect(int x, int y, int width, int height, Color color) {
    for (int row = y; row < y + height; row++) {
        for (int col = x; col < x + width; col++) {
            put_pixel(col, row, color);
        }
    }
}

// Draw rectangle outline
void draw_rect(int x, int y, int width, int height, Color color) {
    // Top and bottom edges
    for (int col = x; col < x + width; col++) {
        put_pixel(col, y, color);
        put_pixel(col, y + height - 1, color);
    }
    // Left and right edges
    for (int row = y; row < y + height; row++) {
        put_pixel(x, row, color);
        put_pixel(x + width - 1, row, color);
    }
}

// Clear screen
void clear_screen(Color color) {
    fill_rect(0, 0, fb.width, fb.height, color);
}
```

## Line Drawing (Bresenham's Algorithm)

```c
void draw_line(int x0, int y0, int x1, int y1, Color color) {
    int dx = abs(x1 - x0);
    int dy = abs(y1 - y0);
    int sx = (x0 < x1) ? 1 : -1;
    int sy = (y0 < y1) ? 1 : -1;
    int err = dx - dy;
    
    while (1) {
        put_pixel(x0, y0, color);
        
        if (x0 == x1 && y0 == y1) break;
        
        int e2 = 2 * err;
        if (e2 > -dy) {
            err -= dy;
            x0 += sx;
        }
        if (e2 < dx) {
            err += dx;
            y0 += sy;
        }
    }
}
```

## Circle Drawing (Midpoint Circle Algorithm)

```c
void draw_circle(int cx, int cy, int radius, Color color, int filled) {
    int x = 0;
    int y = radius;
    int d = 3 - 2 * radius;
    
    while (x <= y) {
        if (filled) {
            // Fill vertical lines through circle
            draw_line(cx - x, cy + y, cx + x, cy + y, color);
            draw_line(cx - x, cy - y, cx + x, cy - y, color);
            draw_line(cx - y, cy + x, cx + y, cy + x, color);
            draw_line(cx - y, cy - x, cx + y, cy - x, color);
        } else {
            // Draw 8 octants
            put_pixel(cx + x, cy + y, color);
            put_pixel(cx - x, cy + y, color);
            put_pixel(cx + x, cy - y, color);
            put_pixel(cx - x, cy - y, color);
            put_pixel(cx + y, cy + x, color);
            put_pixel(cx - y, cy + x, color);
            put_pixel(cx + y, cy - x, color);
            put_pixel(cx - y, cy - x, color);
        }
        
        if (d < 0) {
            d = d + 4 * x + 6;
        } else {
            d = d + 4 * (x - y) + 10;
            y--;
        }
        x++;
    }
}
```

## Text Rendering

```c
// 8x8 monospace bitmap font
struct Font {
    uint8_t width;
    uint8_t height;
    uint8_t glyphs[256][64];       // 256 chars, 8x8 pixels
};

// Draw single character
void draw_char(int x, int y, char ch, Color color, struct Font *font) {
    uint8_t *glyph = font->glyphs[(unsigned char)ch];
    
    for (int row = 0; row < 8; row++) {
        uint8_t bits = glyph[row];
        for (int col = 0; col < 8; col++) {
            if (bits & (1 << (7 - col))) {
                put_pixel(x + col, y + row, color);
            }
        }
    }
}

// Draw text string
void draw_text(int x, int y, const char *text, Color color, struct Font *font) {
    int pos_x = x;
    for (const char *ch = text; *ch; ch++) {
        if (*ch == '\n') {
            y += 8;
            pos_x = x;
        } else {
            draw_char(pos_x, y, *ch, color, font);
            pos_x += 8;
        }
    }
}
```

## Double Buffering (Optional)

```c
// Back buffer for smooth animations
void *back_buffer;

void init_double_buffering() {
    back_buffer = kmalloc(fb.total_size);
}

// Use back buffer for drawing
void put_pixel_buffered(int x, int y, Color color) {
    if (x < 0 || x >= fb.width || y < 0 || y >= fb.height) return;
    
    uint32_t offset = y * fb.pitch + x * fb.bytes_per_pixel;
    *(Color*)((uint8_t*)back_buffer + offset) = color;
}

// Copy back buffer to screen
void flip_buffer() {
    memcpy(fb.base_address, back_buffer, fb.total_size);
}
```

---

**Related Documentation:**
- [ARCHITECTURE.md](../ARCHITECTURE.md)
- [docs/UI_ARCHITECTURE.md](UI_ARCHITECTURE.md)