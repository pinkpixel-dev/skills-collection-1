---
name: gpui-builder
description: >
  Complete guide for building GPU-accelerated desktop UI applications in Rust using GPUI — the hybrid immediate+retained mode framework powering the Zed editor. ALWAYS use this skill when the user mentions GPUI, gpui.rs, Zed UI framework, building a Rust GUI with GPU acceleration, or wants to create desktop apps in Rust with a Tailwind-like styling API. Also trigger when the user asks about Entities, Views, or Elements in a Rust GUI context, or wants to implement keyboard actions, async UI, custom canvas drawing, or testing in GPUI. If the user says "build me a Rust app with GPUI", "I want to use gpui", "how do I make a window in Rust", or asks about Zed's rendering architecture, use this skill immediately. Covers: architecture (Entities/Views/Elements), styling API, event/action system, async integration, custom canvas, testing, packaging, and comparison with iced/egui/slint.
---

# GPUI Builder

GPUI is a **GPU-accelerated, hybrid immediate+retained mode** Rust UI framework developed by the Zed team. It renders the entire window on the GPU like a game engine, using Metal (macOS) or GL/Vulkan (Linux).

## Quick Reference

| Concept | What it is |
|---|---|
| `Entity<T>` | App-owned state (like `Rc<T>`); survives across frames |
| `Render` trait | Implement on your view struct; returns element tree each frame |
| `div()` | Primary building block; styled container, supports children |
| `Application::new().run(...)` | Entry point |
| `cx.open_window(...)` | Creates an OS window |
| `cx.new(|_| MyView {...})` | Registers a view entity |
| `#[gpui::action]` | Defines a keyboard-bindable action |
| `canvas(prepaint, paint)` | Custom GPU drawing element |

## Workflow

1. Add `gpui = "0.2"` to `Cargo.toml`
2. Define state structs (or use `Entity<T>` for shared state)
3. Implement `Render` for your root view
4. Call `Application::new().run(...)` with `cx.open_window(...)`
5. Wire up events via `.on_action()` / `.on_click()` + keymap JSON
6. Use `cx.spawn(async {...})` for async work

## Platform Setup

- **macOS**: Requires Xcode + command-line tools. Uses Metal.
- **Linux**: Needs X11/Wayland dev libs (`libxcb`, `libxcb-icccm`, etc.). Uses GL/Vulkan.
- **Windows**: Not officially supported yet; community `gpui-windows` crate exists.
- **Mobile**: Alpha-quality via separate `gpui_mobile` crate (uses wgpu).

## Hello World — Minimal App

```rust
use gpui::{prelude::*, App, Application, WindowOptions, WindowBounds, Bounds, size, px, div, rgb};

struct HelloWorld { text: SharedString }

impl Render for HelloWorld {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex().flex_col().gap_3()
            .bg(rgb(0x505050))
            .size(px(500.), px(500.))
            .justify_center().items_center()
            .shadow_lg().border_1().border_color(rgb(0x0000ff))
            .text_xl().text_color(rgb(0xffffff))
            .child(format!("Hello, {}!", &self.text))
    }
}

fn main() {
    Application::new().run(|cx: &mut App| {
        let bounds = Bounds::centered(None, size(px(500.), px(500.)), cx);
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            |_window, cx| cx.new(|_| HelloWorld { text: "World".into() }),
        ).unwrap();
    });
}
```

## Styling API (Tailwind-like)

Chain methods directly on elements — no CSS files:

```rust
div()
    .flex()                          // display: flex
    .flex_col()                      // flex-direction: column
    .gap_3()                         // gap: 0.75rem
    .justify_center()                // justify-content: center
    .items_center()                  // align-items: center
    .bg(rgb(0x1a1a2e))               // background color
    .border_1()                      // border-width: 1px
    .border_color(rgb(0x4444ff))     // border color
    .padding_4()                     // padding: 1rem
    .text_xl()                       // font-size: xl
    .text_color(rgb(0xffffff))       // color: white
    .shadow_lg()                     // box-shadow: large
    .overflow_auto()                 // overflow: auto
    .size(px(400.), px(300.))        // width + height
    .child("some text or element")
```

Built-in color helpers: `gpui::red()`, `gpui::green()`, `gpui::blue()`, `gpui::yellow()`, `gpui::black()`, `gpui::white()`

## Keyboard Actions System

GPUI is keyboard-first. Actions decouple keys from logic:

```rust
// 1. Define actions
#[gpui::action] struct MoveUp;
#[gpui::action] struct MoveDown;
#[gpui::action] struct Submit;

// 2. Attach handlers in render()
impl Render for MyMenu {
    fn render(&mut self, _win: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .key_context("menu")   // scopes the keymap
            .on_action(|this: &mut MyMenu, _: &MoveUp, _win, _cx| {
                this.selection -= 1;
            })
            .on_action(|this, _: &MoveDown, _win, _cx| {
                this.selection += 1;
            })
            .children(/* render items */)
    }
}
```

```json
// 3. Keymap JSON (load via cx.bind_keys or keymap file)
{
  "context": "menu",
  "bindings": {
    "up": "my_app::MoveUp",
    "down": "my_app::MoveDown",
    "enter": "my_app::Submit"
  }
}
```

## Async Integration

GPUI ships its own async executor — no tokio required (but you can mix):

```rust
// Spawn async work from a click handler or anywhere you have cx
cx.spawn(async move |cx| {
    let data = fetch_data().await;
    cx.update(|cx| {
        cx.get_entity(&my_entity).unwrap().data = data;
    });
});

// Or get an AsyncApp handle for 'static contexts
let async_app: AsyncApp = cx.to_async();
```

## Entity / State Management

```rust
// Create shared state
let list: Entity<Vec<String>> = cx.new_entity(Vec::new());
let input: Entity<SharedString> = cx.new_entity(SharedString::new());

// Access in render (read)
let items = list.read(cx);

// Mutate
list.update(cx, |items, _cx| {
    items.push("new item".into());
});
```

## Custom Canvas (Advanced)

For custom drawing (charts, paint tools, game overlays):

```rust
use gpui::{canvas, Bounds, Window};

fn render(&mut self, _win: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    let color = self.accent_color; // capture state for paint closure

    canvas(
        // prepaint: runs before paint, returns data T passed to paint
        move |_bounds, _win, _cx| color,
        // paint: issue GPU draw calls
        move |bounds, color, win, _cx| {
            let ctx = win.begin_draw(bounds);
            ctx.fill_path(
                gpui::path().rect(bounds).finish(),
                color,
            );
        }
    )
    .size(px(400.), px(400.))
}
```

## Testing

```rust
#[gpui::test]
fn test_my_view() {
    let mut app = TestAppContext::new();
    let view = app.open_window(Default::default(), |_, cx| {
        cx.new(|_| MyView { count: 0 })
    }).unwrap();

    // Simulate interactions
    app.simulate_keydown("enter", Modifiers::none());

    // Assert on view state
    view.read_with(&app, |view, _| {
        assert_eq!(view.count, 1);
    });
}
```

## Common Gotchas

- **Single root element**: `render()` must return exactly one root element. Wrap siblings in a parent `div()`.
- **Breaking changes**: GPUI is pre-1.0 — pin your version and check CHANGELOG on updates.
- **No built-in widgets**: No `Button`, `Label`, etc. in core. Build from `div()` or use community `gpui-component` crate.
- **Text input**: Requires manual `KeyDown`/`handle_input` wiring. See the official Input example at gpui.rs.
- **Windows target**: Not officially supported yet; may not compile without community patches.
- **Don't block render()**: Never do heavy CPU work inside `render()`. Use `cx.spawn(async {...})` for I/O.

## Deeper Reference

For detailed examples and comparisons, see:
- `references/architecture.md` — Entity/View/Element system deep dive
- `references/comparison.md` — GPUI vs iced vs egui vs Slint comparison table
- `references/examples.md` — Full dynamic list + input example, advanced canvas example

## Key Resources

- **Official docs**: https://docs.rs/gpui/latest/gpui/
- **Source + examples**: https://github.com/zed-industries/zed/tree/main/crates/gpui
- **Website**: https://gpui.rs/
- **Community widget lib**: https://github.com/longbridge/gpui-component
- **Mobile port**: https://github.com/itsbalamurali/gpui-mobile
- **Discord**: Zed community Discord (search `#gpui`)
