# GPUI Full Examples

## Dynamic List with Text Input

A complete working example: text input + "Add" button + scrollable list of items.

```rust
use gpui::{
    prelude::*, App, Application, Context, Entity, SharedString,
    WindowOptions, WindowBounds, Bounds, Window, div, px, size, rgb,
};

struct ListApp {
    items: Entity<Vec<SharedString>>,
    input:  Entity<SharedString>,
}

impl ListApp {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            items: cx.new_entity(Vec::new()),
            input: cx.new_entity(SharedString::default()),
        }
    }
}

impl Render for ListApp {
    fn render(&mut self, _win: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let items    = self.items.read(cx).clone();
        let input_val = self.input.read(cx).clone();

        let items_entity = self.items.clone();
        let input_entity  = self.input.clone();

        // Add button handler — clones needed to move into closure
        let items_clone = items_entity.clone();
        let input_clone = input_entity.clone();

        div()
            .flex().flex_col().gap_3()
            .padding_4()
            .bg(rgb(0x1a1a2e))
            .size_full()
            // ── Input row ──────────────────────────────────
            .child(
                div().flex().gap_2().items_center()
                    // Text input field
                    .child(
                        div()
                            .flex_1()
                            .border_1().border_color(rgb(0x444466))
                            .bg(rgb(0x2a2a3e))
                            .padding_2()
                            .text_color(rgb(0xffffff))
                            .child(input_val.clone())
                            // Handle keyboard text input
                            .on_key_down(move |_this, ev, _win, cx| {
                                if let Some(ch) = ev.keystroke.key.chars().next() {
                                    if ev.keystroke.key == "backspace" {
                                        input_entity.update(cx, |s, _| {
                                            let mut owned = s.to_string();
                                            owned.pop();
                                            *s = owned.into();
                                        });
                                    } else if !ev.keystroke.modifiers.command {
                                        input_entity.update(cx, |s, _| {
                                            let mut owned = s.to_string();
                                            owned.push(ch);
                                            *s = owned.into();
                                        });
                                    }
                                }
                            })
                    )
                    // Add button
                    .child(
                        div()
                            .px_4().py_2()
                            .bg(rgb(0x4444cc))
                            .text_color(rgb(0xffffff))
                            .rounded_md()
                            .cursor_pointer()
                            .on_click(move |_this, _ev, _win, cx| {
                                let text = input_clone.read(cx).clone();
                                if !text.is_empty() {
                                    items_clone.update(cx, |list, _| list.push(text));
                                    input_clone.update(cx, |s, _| *s = SharedString::default());
                                }
                            })
                            .child("Add")
                    )
            )
            // ── Item list ──────────────────────────────────
            .child(
                div()
                    .flex().flex_col().gap_1()
                    .overflow_auto()
                    .flex_1()
                    .children(items.iter().enumerate().map(|(i, item)| {
                        let item_entity = items_entity.clone();
                        div()
                            .flex().items_center().justify_between()
                            .px_3().py_2()
                            .bg(rgb(0x2a2a3e))
                            .rounded_md()
                            .text_color(rgb(0xddddff))
                            .child(item.clone())
                            // Remove button per item
                            .child(
                                div()
                                    .px_2().py_1()
                                    .bg(rgb(0x662222))
                                    .text_color(rgb(0xffaaaa))
                                    .rounded_sm()
                                    .cursor_pointer()
                                    .on_click(move |_, _, _, cx| {
                                        item_entity.update(cx, |list, _| {
                                            list.remove(i);
                                        });
                                    })
                                    .child("×")
                            )
                    }))
            )
    }
}

fn main() {
    Application::new().run(|cx: &mut App| {
        let bounds = Bounds::centered(None, size(px(600.), px(500.)), cx);
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            |_win, cx| cx.new(|cx| ListApp::new(cx)),
        ).unwrap();
    });
}
```

---

## Custom Canvas: Animated Graph

Draw a sine wave chart that updates each frame using GPUI's canvas element:

```rust
use gpui::{canvas, prelude::*, px, rgb, App, Application, Context,
           Bounds, Window, WindowOptions, WindowBounds, size};
use std::f32::consts::PI;

struct GraphView {
    tick: f32,
}

impl Render for GraphView {
    fn render(&mut self, _win: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let tick = self.tick;
        self.tick += 0.05; // advance each frame

        canvas(
            move |_bounds, _win, _cx| tick,
            move |bounds, tick, win, _cx| {
                let ctx = win.begin_draw(bounds);
                let w = bounds.size.width.0;
                let h = bounds.size.height.0;

                // Background
                ctx.fill_path(
                    gpui::path().rect(bounds).finish(),
                    rgb(0x0d0d1a),
                );

                // Draw sine wave
                let mut path = gpui::path();
                let steps = 200usize;
                for i in 0..=steps {
                    let x = (i as f32 / steps as f32) * w;
                    let y = h / 2.0 + (((i as f32 / steps as f32) * 4.0 * PI) + tick).sin() * (h / 3.0);
                    if i == 0 {
                        path = path.move_to((x, y));
                    } else {
                        path = path.line_to((x, y));
                    }
                }

                ctx.stroke_path(
                    path.finish(),
                    gpui::StrokeStyle::solid(rgb(0x44aaff), 2.0),
                );
            }
        )
        .size(px(600.), px(300.))
    }
}

fn main() {
    Application::new().run(|cx: &mut App| {
        let bounds = Bounds::centered(None, size(px(620.), px(340.)), cx);
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            |_win, cx| cx.new(|_| GraphView { tick: 0.0 }),
        ).unwrap();
    });
}
```

---

## Multi-Panel Layout

Sidebar + main content area pattern:

```rust
fn render(&mut self, _win: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    div()
        .flex()                         // horizontal flex
        .size_full()
        .bg(rgb(0x111120))
        // Sidebar
        .child(
            div()
                .w(px(220.))
                .h_full()
                .flex().flex_col().gap_1()
                .padding_3()
                .bg(rgb(0x1a1a2e))
                .border_r_1().border_color(rgb(0x333355))
                .children(self.nav_items.iter().map(|item| {
                    let active = item.id == self.active_item;
                    div()
                        .px_3().py_2()
                        .rounded_md()
                        .text_color(if active { rgb(0xffffff) } else { rgb(0x8888aa) })
                        .bg(if active { rgb(0x3333aa) } else { rgb(0x000000) })
                        .cursor_pointer()
                        .child(item.label.clone())
                }))
        )
        // Main area
        .child(
            div()
                .flex_1()
                .h_full()
                .flex().flex_col()
                .padding_6()
                .text_color(rgb(0xccccee))
                .child(/* main content */)
        )
}
```

---

## Async Data Loading Pattern

```rust
struct DataView {
    data:    Entity<Option<String>>,
    loading: Entity<bool>,
}

impl DataView {
    fn fetch(&self, cx: &mut Context<Self>) {
        let data_entity    = self.data.clone();
        let loading_entity = self.loading.clone();

        loading_entity.update(cx, |v, _| *v = true);

        cx.spawn(async move |cx| {
            // Simulated async fetch
            let result = do_http_request().await.ok();

            cx.update(|cx| {
                data_entity.update(cx, |d, _| *d = result);
                loading_entity.update(cx, |v, _| *v = false);
            }).ok();
        }).detach();
    }
}

impl Render for DataView {
    fn render(&mut self, _win: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let loading = *self.loading.read(cx);
        let data    = self.data.read(cx).clone();

        div()
            .flex().flex_col().gap_4()
            .padding_6()
            .child(if loading {
                div().text_color(rgb(0xaaaaff)).child("Loading...")
            } else if let Some(text) = data {
                div().text_color(rgb(0xffffff)).child(text)
            } else {
                div().text_color(rgb(0xff4444)).child("No data")
            })
    }
}
```
