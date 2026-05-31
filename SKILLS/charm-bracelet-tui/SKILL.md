---
name: charm-bracelet-tui
description: >
  Complete guide for building beautiful, interactive Go TUI (terminal UI) applications using the Charm Bracelet ecosystem. ALWAYS use this skill when the user mentions: bubbletea, bubbles, lipgloss, glamour, huh, harmonica, bubblezone, ntcharts, TUI in Go, terminal user interface in Go, terminal app with Go, CLI with interactive UI, charm.sh, charm.land, or wants to build any kind of interactive terminal application. Also trigger when building Go CLI tools that need forms, menus, progress bars, spinners, charts, markdown rendering, styled output, or mouse support. This covers the full stack from framework to styling to widgets to animations.
---

# Charm Bracelet TUI Stack

The Charm Bracelet ecosystem is a collection of Go libraries for building beautiful terminal UIs. They are designed to work together but can also be used independently.

**Current versions:** All core Charm libs are on **v2** as of early 2026 (import paths use `charm.land/<lib>/v2`).

## The Stack at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│                   Your TUI Application                       │
├──────────────────────────┬──────────────────────────────────┤
│   huh (forms/prompts)    │   Bubbles (pre-built components)  │
├──────────────────────────┴──────────────────────────────────┤
│                BubbleTea (MVU framework)                     │
├──────────────────┬──────────────┬───────────────────────────┤
│  Lip Gloss       │   Glamour    │   Harmonica               │
│  (styling)       │  (markdown)  │   (animation)             │
├──────────────────┴──────────────┴───────────────────────────┤
│           BubbleZone (mouse tracking) │ ntcharts (charts)    │
└─────────────────────────────────────────────────────────────┘
```

## Library Roles

| Library | Role | Import |
|---------|------|--------|
| **BubbleTea** | MVU framework — the backbone of every app | `charm.land/bubbletea/v2` |
| **Bubbles** | Pre-built TUI components (inputs, lists, etc.) | `charm.land/bubbles/v2` (note: check go.dev for exact path) |
| **Lip Gloss** | Styling, layout, colors, borders, tables, trees | `charm.land/lipgloss/v2` |
| **Glamour** | Stylesheet-based Markdown renderer | `github.com/charmbracelet/glamour` |
| **Huh** | Terminal forms and interactive prompts | `charm.land/huh/v2` |
| **Harmonica** | Physics-based spring animations | `github.com/charmbracelet/harmonica` |
| **BubbleZone** | Mouse event tracking across components | `github.com/lrstanley/bubblezone/v2` |
| **ntcharts** | Terminal charts (bar, line, sparkline, heatmap, etc.) | `github.com/NimbleMarkets/ntcharts` |

## Quick Reference: Which Library for What?

- **Need a TUI framework?** → BubbleTea (always the foundation)
- **Need forms / user input collection?** → Huh (or Bubbles textinput/textarea)
- **Need styled output / layouts?** → Lip Gloss
- **Need to render Markdown?** → Glamour
- **Need pre-built components?** → Bubbles (spinner, progress, list, table, viewport, filepicker...)
- **Need smooth animations?** → Harmonica
- **Need mouse click detection per-component?** → BubbleZone
- **Need charts / graphs?** → ntcharts

---

## How They Work Together

### BubbleTea + Lip Gloss (the core combo)
BubbleTea handles the update loop; Lip Gloss handles the rendering. Your `View()` method returns a `tea.View` built by rendering Lip Gloss styles:

```go
func (m model) View() tea.View {
    title := lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("#FF75B7")).Render("My App")
    body := lipgloss.NewStyle().Padding(1, 2).Render("Hello world")
    return tea.NewView(lipgloss.JoinVertical(lipgloss.Left, title, body))
}
```

### BubbleTea + Huh (embedded forms)
`huh.Form` is a `tea.Model`, so embed it directly:

```go
type Model struct {
    form *huh.Form
}
func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    form, cmd := m.form.Update(msg)
    if f, ok := form.(*huh.Form); ok { m.form = f }
    return m, cmd
}
func (m Model) View() string { return m.form.View() }
```

### BubbleTea + BubbleZone (mouse support)
Wrap your root `View()` in `zone.Scan()`, mark individual areas with `zone.Mark(id, content)`, check `zone.Get(id).InBounds(mouseMsg)` in `Update()`.

### BubbleTea + ntcharts (charts)
ntcharts components are standalone (they have their own `Draw()`/`View()` methods), so embed them in your model and call them during `Update()` and `View()`.

---

## Getting Started

```bash
# Core framework
go get charm.land/bubbletea/v2

# Styling
go get charm.land/lipgloss/v2

# Pre-built components
go get github.com/charmbracelet/bubbles

# Forms
go get charm.land/huh/v2

# Markdown
go get github.com/charmbracelet/glamour

# Animations
go get github.com/charmbracelet/harmonica

# Mouse tracking
go get github.com/lrstanley/bubblezone/v2

# Charts
go get github.com/NimbleMarkets/ntcharts
```

---

## Minimal BubbleTea App (v2 API)

```go
package main

import (
    "fmt"
    "os"
    tea "charm.land/bubbletea/v2"
)

type model struct{ count int }

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "q", "ctrl+c": return m, tea.Quit
        case "up": m.count++
        case "down": m.count--
        }
    }
    return m, nil
}

func (m model) View() tea.View {
    return tea.NewView(fmt.Sprintf("Count: %d\n\nPress q to quit.", m.count))
}

func main() {
    p := tea.NewProgram(model{})
    if _, err := p.Run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

---

## Reference Files

Read the relevant reference file(s) for deep implementation details:

- `references/bubbletea.md` — BubbleTea framework: MVU lifecycle, commands, messages, v2 View API
- `references/lipgloss.md` — Lip Gloss: styles, colors, layouts, tables, trees, lists, compositing
- `references/bubbles.md` — Bubbles components: spinner, textinput, textarea, list, table, viewport, progress, filepicker, timer, help
- `references/glamour.md` — Glamour: markdown rendering, themes, custom styles
- `references/huh.md` — Huh: forms, fields, dynamic forms, BubbleTea integration, spinner
- `references/harmonica.md` — Harmonica: spring animations, damping ratios, BubbleTea integration
- `references/bubblezone.md` — BubbleZone: mouse zones, InBounds, global vs local manager
- `references/ntcharts.md` — ntcharts: canvas, bar, line, sparkline, heatmap, OHLC, time series, waveline

### When to Read Which

| Task | Reference(s) |
|------|-------------|
| Building a new TUI app from scratch | `bubbletea.md` → `lipgloss.md` |
| Adding forms/prompts | `huh.md` |
| Adding pre-built widgets | `bubbles.md` |
| Rendering markdown | `glamour.md` |
| Adding animations | `harmonica.md` |
| Mouse click detection | `bubblezone.md` |
| Charts and data visualization | `ntcharts.md` |
| Complex layouts / styling | `lipgloss.md` |

---

## Common Gotchas

- **v2 import paths**: Core Charm libs moved to `charm.land/<lib>/v2` in 2025-2026. `github.com/charmbracelet/*` paths still work for some (glamour, harmonica), but prefer `charm.land` for bubbletea, lipgloss, huh.
- **BubbleTea v2 View**: `View()` now returns `tea.View` (not `string`). Use `tea.NewView(str)` to wrap string content.
- **AltScreen for BubbleZone**: BubbleZone requires alt-screen mode — set `view.AltScreen = true` in your root view.
- **Dynamic hex colors in Lip Gloss**: Use `lipgloss.Color("#RRGGBB")` — Lip Gloss will automatically downsample for lower-color terminals.
- **Lip Gloss Width vs len()**: Always use `lipgloss.Width(str)` instead of `len()` — ANSI codes inflate `len()`.
- **BubbleZone zone.Scan() at root only**: Only wrap your root model's view output with `zone.Scan()`.
- **ntcharts**: Call `.Draw()` before `.View()` to rasterize the chart each frame.
