# BubbleTea Reference

**Repo:** https://github.com/charmbracelet/bubbletea  
**Docs:** https://pkg.go.dev/charm.land/bubbletea/v2  
**Import:** `charm.land/bubbletea/v2`  
**Version:** v2.x (latest as of 2026)

---

## The Elm Architecture (MVU)

BubbleTea is based on The Elm Architecture:
- **Model** — application state (any struct)
- **Update** — handles events, returns updated model + optional commands
- **View** — renders the UI from the current model

```go
type tea.Model interface {
    Init() tea.Cmd
    Update(tea.Msg) (tea.Model, tea.Cmd)
    View() tea.View   // v2: returns tea.View, not string
}
```

---

## v2 View API

In v2, `View()` returns `tea.View` instead of `string`.

```go
func (m model) View() tea.View {
    // Wrap string content
    v := tea.NewView("hello world")
    
    // Or build from options
    v = tea.NewView(content,
        tea.WithAltScreen(true),         // enable alt screen
        tea.WithMouseMode(tea.MouseModeCellMotion), // mouse tracking
    )
    
    // Or set properties directly
    var view tea.View
    view.AltScreen = true
    view.MouseMode = tea.MouseModeCellMotion
    view.SetContent(renderedString)
    return view
}
```

Mouse modes:
- `tea.MouseModeDisabled` — no mouse
- `tea.MouseModeNormal` — basic mouse (clicks, wheel)
- `tea.MouseModeCellMotion` — full motion tracking (hover)

---

## Program Setup

```go
// Basic
p := tea.NewProgram(initialModel())
if _, err := p.Run(); err != nil { ... }

// With options
p := tea.NewProgram(
    initialModel(),
    tea.WithAltScreen(),       // full-screen mode
    tea.WithMouseCellMotion(), // enable mouse
)

// Run returns final model
finalModel, err := p.Run()
```

---

## Messages

Messages are the "something happened" signal. They can be any Go type.

```go
// Key presses (v2)
case tea.KeyPressMsg:
    switch msg.String() {
    case "ctrl+c", "q": return m, tea.Quit
    case "up", "k":     m.cursor--
    case "enter", " ":  // toggle selection
    }

// Key release (v2 new)
case tea.KeyReleaseMsg:
    // handle key release

// Mouse (v2)
case tea.MouseClickMsg:
    // msg.Button: tea.MouseLeft, tea.MouseRight, tea.MouseMiddle
    // msg.X, msg.Y: terminal coordinates

case tea.MouseReleaseMsg:
    // same fields as ClickMsg

case tea.MouseMotionMsg:
    // requires MouseModeCellMotion

case tea.MouseWheelMsg:
    // msg.Button: tea.MouseWheelUp, tea.MouseWheelDown

// Window resize
case tea.WindowSizeMsg:
    m.width, m.height = msg.Width, msg.Height

// Background color (for adaptive theming with Lip Gloss)
case tea.BackgroundColorMsg:
    m.styles = newStyles(msg.IsDark())
```

---

## Commands

Commands are functions that perform I/O and return a message.

```go
// Built-in commands
tea.Quit                  // quit the program
tea.EnterAltScreen        // enter alt screen
tea.ExitAltScreen         // exit alt screen
tea.RequestBackgroundColor // request terminal bg color

// Batch multiple commands
return m, tea.Batch(cmd1, cmd2, cmd3)

// Sequence commands (run one after another)
return m, tea.Sequence(cmd1, cmd2)

// Custom command (returns a Msg)
func fetchData() tea.Msg {
    // do I/O
    return DataMsg{data: result}
}
return m, fetchData // Note: pass function, not call

// Tick (for timers/animation)
return m, tea.Every(time.Second, func(t time.Time) tea.Msg {
    return tickMsg(t)
})

// Or use tea.Tick for one-shot
return m, tea.Tick(100*time.Millisecond, func(t time.Time) tea.Msg {
    return tickMsg(t)
})
```

---

## Common Patterns

### Handling Window Resize

```go
type model struct {
    width, height int
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.width = msg.Width
        m.height = msg.Height
    }
    return m, nil
}
```

### Composing Sub-Models (components)

```go
type parentModel struct {
    child childModel
}

func (m parentModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    m.child, cmd = m.child.Update(msg)
    return m, cmd
}

func (m parentModel) View() tea.View {
    return tea.NewView(m.child.View())
}
```

### Spinner Animation Example

```go
type model struct {
    spinner  spinner.Model
    loading  bool
}

func (m model) Init() tea.Cmd {
    return m.spinner.Tick  // start ticking
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    m.spinner, cmd = m.spinner.Update(msg)
    return m, cmd
}
```

### Alt Screen with Cleanup

```go
p := tea.NewProgram(model{}, tea.WithAltScreen())
finalModel, err := p.Run()
// alt screen automatically restored on exit
```

### Executing External Commands

```go
// Run an external program (pauses BubbleTea, runs editor, resumes)
return m, tea.ExecProcess(exec.Command("vim", filename), func(err error) tea.Msg {
    return editorFinishedMsg{err}
})
```

### Debug Logging

```go
// Log to file (can't log to stdout in a TUI)
if debug := os.Getenv("DEBUG"); debug != "" {
    f, _ := tea.LogToFile("debug.log", "debug")
    defer f.Close()
}
```

---

## Key String Reference

Common key strings for `tea.KeyPressMsg.String()`:

| Key | String |
|-----|--------|
| Arrow up | `"up"` |
| Arrow down | `"down"` |
| Arrow left | `"left"` |
| Arrow right | `"right"` |
| Enter | `"enter"` |
| Space | `" "` |
| Tab | `"tab"` |
| Backspace | `"backspace"` |
| Escape | `"esc"` |
| Ctrl+C | `"ctrl+c"` |
| Ctrl+D | `"ctrl+d"` |
| F1–F12 | `"f1"` – `"f12"` |
| Letters/nums | `"a"`, `"1"`, etc. |
| Shift+letter | `"A"`, `"B"`, etc. |

---

## Upgrade from v1

Key changes in v2:
- `View()` returns `tea.View` instead of `string`
- Use `tea.NewView(str)` to wrap strings
- `tea.KeyMsg` → `tea.KeyPressMsg` and `tea.KeyReleaseMsg`
- `tea.MouseMsg` split into `tea.MouseClickMsg`, `tea.MouseReleaseMsg`, `tea.MouseMotionMsg`, `tea.MouseWheelMsg`
- Mouse mode set on `tea.View` instead of program option (though `tea.WithMouseCellMotion()` still works)

Full upgrade guide: https://github.com/charmbracelet/bubbletea/blob/main/UPGRADE_GUIDE_V2.md
