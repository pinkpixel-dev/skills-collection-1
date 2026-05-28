# GPUI Architecture Deep Dive

## The Three-Layer Model

```
App (AppContext)
├── Entities (state registry — Entity<T>)
│   └── Views (Entities that implement Render)
│       └── Element Tree (built each frame in render())
│           └── GPU Rasterizer → Screen
└── Window (OS window + event loop)
```

### Entities

`Entity<T>` is GPUI's fundamental state container — think `Rc<RefCell<T>>` but managed by the app context.

- Created via `cx.new_entity(initial_value)` or `cx.new(|cx| MyView { ... })`
- Accessed via `.read(cx)` (immutable) or `.update(cx, |val, cx| { ... })` (mutable)
- Entities owned by `AppContext` — they live until explicitly dropped
- A **View** is just an `Entity<T>` where `T: Render`

```rust
// Plain state entity
let counter: Entity<u32> = cx.new_entity(0u32);

// Mutate
counter.update(cx, |val, _cx| *val += 1);

// Read
let n = *counter.read(cx);
```

### Views (Render Trait)

Any struct implementing `Render` becomes a "view". GPUI calls `render()` every frame to rebuild the element tree.

```rust
pub trait Render: Sized {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement;
}
```

The `Context<Self>` gives you:
- Access to other entities
- Ability to spawn tasks
- Ability to subscribe to entity changes
- `cx.notify()` to request a re-render

### Elements

Elements are the low-level building blocks — they know how to lay out and paint themselves.

- `div()` — the workhorse; a flex container
- `svg()` — renders SVG content
- `img()` — renders images
- `canvas(prepaint, paint)` — custom GPU drawing
- `uniform_list(...)` — virtualized scrollable list (for large datasets)

Custom elements: implement `Element` or `RenderOnce` traits for full control.

## Rendering Model: Hybrid Immediate + Retained

**Retained**: View structs persist between frames. State (`Entity<T>`) persists. You don't rebuild the world from scratch.

**Immediate**: Each frame, `render()` is called and builds a *fresh* element tree from current state. No manual "dirty flags" — if state changes, GPUI re-renders.

**GPU**: The entire built element tree is submitted to the GPU (Metal/GL) for rasterization every frame. No CPU blitting.

## Event Flow

```
OS Event (key/mouse) 
  → Window event loop 
  → GPUI dispatch
  → Action resolution (key_context + keymap) 
  → on_action handler closure
  → State mutation via Entity::update()
  → cx.notify() triggers re-render
  → render() called → new element tree
  → GPU draw
```

## Window Lifecycle

```rust
cx.open_window(options, |_window, cx| {
    // This closure returns the root Entity<impl Render>
    cx.new(|_| MyRootView::new())
})
```

You can open multiple windows. Each window has its own root view entity.

Window options:
```rust
WindowOptions {
    window_bounds: Some(WindowBounds::Windowed(bounds)),
    titlebar: Some(TitlebarOptions {
        title: Some("My App".into()),
        appears_transparent: false,
        ..Default::default()
    }),
    focus: true,
    ..Default::default()
}
```

## Subscriptions & Reactivity

Subscribe to entity changes to trigger re-renders:

```rust
// In your view's initialization (e.g. in a new() fn called from cx.new())
impl MyView {
    fn new(cx: &mut Context<Self>) -> Self {
        // Re-render this view when `other_entity` changes
        cx.observe(&other_entity, |this, _entity, cx| {
            cx.notify(); // triggers re-render
        }).detach();

        MyView { other_entity }
    }
}
```

## Async Patterns

```rust
// Pattern 1: spawn from a context (fire and forget)
cx.spawn(async move |cx| {
    let result = some_async_fn().await;
    cx.update(|cx| {
        // update state here
    }).ok();
}).detach();

// Pattern 2: AsyncApp for 'static tasks
let async_cx: AsyncApp = cx.to_async();
tokio::spawn(async move {
    let data = fetch().await;
    async_cx.update(|cx| {
        my_entity.update(cx, |state, _| state.data = data);
    }).ok();
});
```

## Inspector / Debug Mode

GPUI includes a built-in UI inspector (similar to browser DevTools). Toggle via a debug build flag or keyboard shortcut. Click elements to inspect their layout properties and style.

## Performance Internals

- Element trees are built and discarded each frame (very fast — Rust allocation)
- GPU batches draw calls — multiple elements can share a draw call
- Text is rendered with vector fonts directly on GPU (no CPU rasterization)
- Off-screen elements are culled before submission
- `uniform_list` virtualizes long lists (only renders visible items)
- Metal command buffers (macOS) or GL display lists (Linux) submitted async from main thread
