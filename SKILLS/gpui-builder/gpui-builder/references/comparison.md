# GPUI vs Other Rust GUI Frameworks

## Quick Decision Matrix

| Need | Recommended |
|---|---|
| Blazing fast GPU desktop app (like an editor/IDE) | **GPUI** |
| Cross-platform app incl. Windows/Web today | **Iced** or **egui** |
| Game overlay / dev tools / immediate mode | **egui** |
| Qt-like declarative UI with designer support | **Slint** |
| Embedded / resource-constrained device UI | **Slint** |
| Web + desktop from same codebase | **Dioxus** or **egui** |

---

## Detailed Comparison

| Feature | GPUI | Iced | egui | Slint |
|---|---|---|---|---|
| **UI Model** | Hybrid retained+immediate (Views + Elements) | Reactive Elm-style (State + Message + update + view) | Immediate mode (paint every frame) | Retained DSL-based (QML-like) |
| **GPU Backend** | Metal (macOS) / GL+Vulkan (Linux) — custom shaders | wgpu or tiny-skia (software) | wgpu / OpenGL / WebGL | Skia / custom GPU renderer |
| **Windows support** | ❌ Community only | ✅ Official | ✅ Official | ✅ Official |
| **macOS support** | ✅ Official (Metal) | ✅ Official | ✅ Official | ✅ Official |
| **Linux support** | ✅ Official | ✅ Official | ✅ Official | ✅ Official |
| **Web (WASM)** | ❌ No | ✅ Yes | ✅ Yes (eframe) | ✅ Yes |
| **Mobile** | ⚠️ Alpha (gpui_mobile) | ❌ Partial | ❌ No | ✅ Yes (embedded) |
| **Styling API** | Tailwind-like method chains | `Style` structs + themes | Programmatic / `egui::Style` | Declarative DSL (SlintUI) |
| **Built-in widgets** | ❌ Very few (div, svg, canvas) | ✅ Many (button, input, scrollable, etc.) | ✅ Moderate (label, button, slider, text) | ✅ Rich (native-look controls) |
| **Custom drawing** | ✅ canvas() API | ✅ via canvas widget | ✅ via Painter API | ✅ via low-level path API |
| **Async support** | ✅ Built-in executor | ✅ iced_futures | ⚠️ Manual integration | ✅ Signals / futures |
| **Text input** | ⚠️ Manual wiring required | ✅ Text input widget | ✅ text_edit widget | ✅ Built-in |
| **Accessibility** | ❌ Not yet | ⚠️ Partial | ⚠️ Partial | ✅ Better support |
| **Performance ceiling** | 🔥 Very high (whole-window GPU) | ✅ Good | ✅ Good for simple | ✅ Good |
| **Maturity** | Pre-1.0 (0.2.x, active) | ~0.6, maturing | 1.0+, stable | 1.0+, stable |
| **Docs quality** | ⚠️ Incomplete (catching up) | ✅ Good | ✅ Good | ✅ Good |
| **Community size** | Small but growing | Medium | Large | Medium |
| **Real-world usage** | Zed editor (150k+ users) | Cosmic Desktop, games | Rerun, tools, games | Commercial products, devices |
| **License** | Apache-2.0 | MIT | MIT | GPL / commercial |

---

## When to Choose GPUI

**Choose GPUI if:**
- You want **the fastest possible Rust desktop UI** (GPU-native)
- You're building an IDE, text editor, or complex tool (similar to Zed's use case)
- You're on macOS or Linux and don't need Windows yet
- You enjoy building from primitives (no pre-built widget reliance)
- You want Tailwind-like styling in Rust
- The Zed team's architecture appeals to you

**Don't choose GPUI if:**
- You need Windows support today
- You need Web/WASM target
- You need accessible, production-ready widgets out of the box
- You want a stable, docs-complete API (it's still evolving fast)
- You're building a quick prototype tool (egui is faster to iterate)

---

## Migration Notes

### From egui to GPUI
- egui is `immediate` — you call `ui.button("Click")` and react. GPUI separates state (Entities) from rendering (Render trait).
- In GPUI, you build element trees in `render()` rather than calling UI functions that draw immediately.
- No `eframe`-equivalent — you set up your own window with `cx.open_window`.

### From Iced to GPUI
- Iced's `Message` enum pattern maps loosely to GPUI's `#[gpui::action]` system.
- Iced's `view()` returning widget trees is similar to GPUI's `render()` returning elements.
- GPUI doesn't have Iced's `Command` / `Subscription` — use `cx.spawn()` and `cx.observe()` instead.

### From React/Web
- `Entity<T>` ≈ `useState` or a Zustand store
- `impl Render` ≈ a React component's `render()` / JSX
- `.on_action()` ≈ event listeners
- `cx.notify()` ≈ `setState()` triggering re-render
- No virtual DOM diffing — GPUI rebuilds element tree each frame (fast due to GPU)
