# UI & Windowing System Architecture

Event-driven graphical user interface with window manager and widget framework.

## Window Manager

```c
struct Window {
    uint32_t window_id;            // Unique window ID
    int x, y;                      // Position
    int width, height;             // Size
    Color background_color;        // Background
    char title[64];                // Window title
    bool visible;                  // Is visible
    bool focused;                  // Has focus
    
    struct Widget *widgets;        // First widget in window
    struct Widget *focused_widget; // Currently focused widget
    
    void (*on_paint)(struct Window *); // Paint callback
    void (*on_event)(struct Window *, struct Event *); // Event callback
} Window;

// Global window manager
struct WindowManager {
    struct Window *windows[256];   // Window table
    struct Window *active_window;  // Currently active
    uint32_t window_count;
} wm;

// Create window
struct Window *window_create(int x, int y, int width, int height, const char *title) {
    struct Window *win = kmalloc(sizeof(struct Window));
    win->window_id = ++next_window_id;
    win->x = x;
    win->y = y;
    win->width = width;
    win->height = height;
    strcpy(win->title, title);
    win->background_color = DARK_GRAY;
    win->visible = true;
    
    wm.windows[wm.window_count++] = win;
    return win;
}

// Render all windows
void window_manager_render() {
    clear_screen(BLACK);
    
    for (int i = 0; i < wm.window_count; i++) {
        struct Window *win = wm.windows[i];
        if (!win->visible) continue;
        
        // Draw window frame
        fill_rect(win->x, win->y, win->width, win->height, win->background_color);
        draw_rect(win->x, win->y, win->width, win->height, WHITE);
        
        // Draw title bar
        fill_rect(win->x, win->y, win->width, 20, DARK_GRAY);
        draw_text(win->x + 4, win->y + 4, win->title, WHITE, &system_font);
        
        // Draw widgets
        for (struct Widget *w = win->widgets; w; w = w->next) {
            if (w->visible) {
                w->draw(w);
            }
        }
        
        // Paint callback
        if (win->on_paint) {
            win->on_paint(win);
        }
    }
}
```

## Event System

```c
enum EventType {
    EVENT_KEY_PRESS = 1,
    EVENT_KEY_RELEASE = 2,
    EVENT_MOUSE_MOVE = 3,
    EVENT_MOUSE_CLICK = 4,
    EVENT_MOUSE_RELEASE = 5,
    EVENT_WINDOW_CLOSE = 6,
    EVENT_WINDOW_FOCUS = 7,
};

struct KeyEvent {
    uint32_t key_code;             // Keyboard scan code
    uint32_t ascii_code;           // ASCII value
    bool shift;                    // Shift key pressed
    bool ctrl;                     // Ctrl key pressed
    bool alt;                      // Alt key pressed
};

struct MouseEvent {
    int x, y;                      // Mouse position
    uint8_t button;                // Button (1=left, 2=middle, 3=right)
};

struct Event {
    uint32_t type;                 // EventType enum
    uint32_t timestamp;            // When event occurred
    union {
        struct KeyEvent key;
        struct MouseEvent mouse;
    } data;
};

// Event queue
struct {
    struct Event queue[256];
    uint32_t head, tail;
} event_queue;

// Post event to queue
void event_post(struct Event *event) {
    event_queue.queue[event_queue.tail] = *event;
    event_queue.tail = (event_queue.tail + 1) % 256;
}

// Process events
void event_process() {
    while (event_queue.head != event_queue.tail) {
        struct Event *event = &event_queue.queue[event_queue.head];
        event_queue.head = (event_queue.head + 1) % 256;
        
        // Route to active window
        if (wm.active_window && wm.active_window->on_event) {
            wm.active_window->on_event(wm.active_window, event);
        }
    }
}
```

## Widget Framework

```c
struct Widget {
    uint32_t widget_id;            // Unique ID
    int x, y;                      // Position relative to window
    int width, height;             // Size
    char label[64];                // Label text
    bool visible;                  // Is visible
    bool enabled;                  // Can interact
    bool focused;                  // Has focus
    
    // Virtual methods
    void (*draw)(struct Widget *);
    void (*on_event)(struct Widget *, struct Event *);
    void (*on_focus)(struct Widget *);
    void (*on_blur)(struct Widget *);
    
    struct Widget *next;           // Next widget in window
};

// Button widget
struct Button {
    struct Widget base;            // Inherits from Widget
    bool pressed;                  // Currently pressed
    void (*on_click)(struct Button *);
};

// Button draw
void button_draw(struct Widget *w) {
    struct Button *btn = (struct Button *)w;
    Color bg = btn->pressed ? BLUE : GRAY;
    
    fill_rect(w->x, w->y, w->width, w->height, bg);
    draw_rect(w->x, w->y, w->width, w->height, WHITE);
    draw_text(w->x + 4, w->y + 4, w->label, WHITE, &system_font);
}

// Button event handler
void button_on_event(struct Widget *w, struct Event *event) {
    struct Button *btn = (struct Button *)w;
    
    if (event->type == EVENT_MOUSE_CLICK) {
        if (event->data.mouse.x >= w->x && 
            event->data.mouse.x < w->x + w->width &&
            event->data.mouse.y >= w->y && 
            event->data.mouse.y < w->y + w->height) {
            btn->pressed = true;
        }
    } else if (event->type == EVENT_MOUSE_RELEASE) {
        if (btn->pressed && btn->on_click) {
            btn->on_click(btn);
        }
        btn->pressed = false;
    }
}

// Create button
struct Button *button_create(struct Window *win, int x, int y, 
                             int width, int height, const char *label) {
    struct Button *btn = kmalloc(sizeof(struct Button));
    btn->base.widget_id = ++next_widget_id;
    btn->base.x = x;
    btn->base.y = y;
    btn->base.width = width;
    btn->base.height = height;
    btn->base.visible = true;
    btn->base.enabled = true;
    strcpy(btn->base.label, label);
    
    btn->base.draw = button_draw;
    btn->base.on_event = button_on_event;
    
    // Add to window
    btn->base.next = win->widgets;
    win->widgets = (struct Widget *)btn;
    
    return btn;
}
```

## Theme System

```c
struct Theme {
    Color primary_color;
    Color secondary_color;
    Color background_color;
    Color text_color;
    Color border_color;
    char font_name[32];
};

struct Theme current_theme = {
    .primary_color = BLUE,
    .secondary_color = CYAN,
    .background_color = DARK_GRAY,
    .text_color = WHITE,
    .border_color = WHITE,
};

// Load theme from file
void theme_load(const char *filename) {
    // Read theme file and update current_theme
}

// Apply theme globally
void theme_apply() {
    // Redraw all windows with new theme
    for (int i = 0; i < wm.window_count; i++) {
        if (wm.windows[i]) {
            // Queue redraw
        }
    }
}
```

---

**Related Documentation:**
- [ARCHITECTURE.md](../ARCHITECTURE.md)
- [docs/GRAPHICS_PIPELINE.md](GRAPHICS_PIPELINE.md)