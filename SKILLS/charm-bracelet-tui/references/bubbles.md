# Bubbles Reference

**Repo:** https://github.com/charmbracelet/bubbles  
**Docs:** https://pkg.go.dev/github.com/charmbracelet/bubbles  
**Import:** `github.com/charmbracelet/bubbles` (check sub-package imports below)  
**Version:** v2.x

Bubbles provides pre-built TUI components (Bubble Tea models) ready to embed in your app.

---

## Component Overview

| Component | Package | Purpose |
|-----------|---------|---------|
| Spinner | `bubbles/spinner` | Loading indicators |
| TextInput | `bubbles/textinput` | Single-line text input |
| TextArea | `bubbles/textarea` | Multi-line text input |
| List | `bubbles/list` | Scrollable, filterable list |
| Table | `bubbles/table` | Tabular data with scrolling |
| Progress | `bubbles/progress` | Progress bar (animated or static) |
| Viewport | `bubbles/viewport` | Scrollable content pane |
| Paginator | `bubbles/paginator` | Pagination logic and dots UI |
| Timer | `bubbles/timer` | Countdown timer |
| Stopwatch | `bubbles/stopwatch` | Count-up stopwatch |
| FilePicker | `bubbles/filepicker` | Filesystem navigation |
| Help | `bubbles/help` | Auto-generated keybinding help |
| Key | `bubbles/key` | Keybinding management |
| Cursor | `bubbles/cursor` | Cursor component for inputs |

---

## Spinner

```go
import "github.com/charmbracelet/bubbles/spinner"

type model struct {
    spinner spinner.Model
    loading bool
}

func NewModel() model {
    s := spinner.New()
    s.Spinner = spinner.Dot     // Dot, Line, MiniDot, Jump, Pulse, Points, Globe, Moon, Monkey
    s.Style = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))
    return model{spinner: s, loading: true}
}

func (m model) Init() tea.Cmd {
    return m.spinner.Tick
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    m.spinner, cmd = m.spinner.Update(msg)
    return m, cmd
}

func (m model) View() string {
    if m.loading {
        return m.spinner.View() + " Loading..."
    }
    return "Done!"
}
```

---

## TextInput

```go
import "github.com/charmbracelet/bubbles/textinput"

type model struct {
    input textinput.Model
}

func NewModel() model {
    ti := textinput.New()
    ti.Placeholder = "Enter something..."
    ti.Focus()                              // focus on start
    ti.CharLimit = 156
    ti.Width = 20
    ti.EchoMode = textinput.EchoNormal      // or EchoPassword, EchoNone
    return model{input: ti}
}

func (m model) Init() tea.Cmd { return textinput.Blink }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "ctrl+c", "esc": return m, tea.Quit
        }
    }
    m.input, cmd = m.input.Update(msg)
    return m, cmd
}

func (m model) View() string {
    return fmt.Sprintf(
        "What's your name?\n\n%s\n\n%s",
        m.input.View(),
        "(ctrl+c to quit)",
    )
}

// Get value
value := m.input.Value()
```

---

## TextArea

```go
import "github.com/charmbracelet/bubbles/textarea"

ta := textarea.New()
ta.Placeholder = "Write something..."
ta.Focus()
ta.SetWidth(40)
ta.SetHeight(5)
ta.CharLimit = 500

// In Update:
m.ta, cmd = m.ta.Update(msg)

// Get value
value := m.ta.Value()

// Control focus
ta.Focus()
ta.Blur()
```

---

## List

```go
import (
    "github.com/charmbracelet/bubbles/list"
    "github.com/charmbracelet/lipgloss"
)

// Implement list.Item interface
type item struct { title, desc string }
func (i item) Title() string       { return i.title }
func (i item) Description() string { return i.desc }
func (i item) FilterValue() string { return i.title }

// Create list
items := []list.Item{
    item{"Raspberry Pi", "A tiny computer"},
    item{"Arduino", "A microcontroller"},
}

const listWidth = 20
const listHeight = 14

delegate := list.NewDefaultDelegate()
l := list.New(items, delegate, listWidth, listHeight)
l.Title = "My List"
l.SetShowStatusBar(false)
l.SetFilteringEnabled(true)
l.Styles.Title = lipgloss.NewStyle().Background(lipgloss.Color("62"))

type model struct { list list.Model }

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.list.SetWidth(msg.Width)
        return m, nil
    }
    var cmd tea.Cmd
    m.list, cmd = m.list.Update(msg)
    return m, cmd
}

func (m model) View() string { return m.list.View() }

// Get selected item
if i, ok := m.list.SelectedItem().(item); ok {
    fmt.Println("Selected:", i.title)
}
```

---

## Table

```go
import "github.com/charmbracelet/bubbles/table"

columns := []table.Column{
    {Title: "Name",  Width: 20},
    {Title: "Email", Width: 30},
    {Title: "Role",  Width: 10},
}

rows := []table.Row{
    {"Alice", "alice@example.com", "admin"},
    {"Bob",   "bob@example.com",   "user"},
}

t := table.New(
    table.WithColumns(columns),
    table.WithRows(rows),
    table.WithFocused(true),
    table.WithHeight(7),
)

s := table.DefaultStyles()
s.Header = s.Header.Bold(true).Foreground(lipgloss.Color("5"))
s.Selected = s.Selected.Foreground(lipgloss.Color("229")).Background(lipgloss.Color("57"))
t.SetStyles(s)

// In Update:
m.table, cmd = m.table.Update(msg)

// Get selected row
row := m.table.SelectedRow()
```

---

## Progress

```go
import "github.com/charmbracelet/bubbles/progress"

// Gradient fill
bar := progress.New(progress.WithGradient("#5A56E0", "#EE6FF8"))

// Solid fill
bar := progress.New(progress.WithSolidFill("#FFFFFF"))

// Animated (uses Harmonica under the hood)
bar := progress.New(progress.WithDefaultGradient())
type progressMsg float64
func tickCmd() tea.Cmd {
    return tea.Tick(100*time.Millisecond, func(t time.Time) tea.Msg {
        return progressMsg(0.01)
    })
}

// In Update:
case progressMsg:
    if m.progress.Percent() >= 1.0 { return m, tea.Quit }
    cmd := m.progress.IncrPercent(float64(msg))
    return m, tea.Batch(tickCmd(), cmd)

var cmd tea.Cmd
m.progress, cmd = m.progress.Update(msg)

// Set value directly
m.progress.SetPercent(0.5)

// In View:
m.progress.View()  // returns string with the bar rendered at bar.Width
m.progress.ViewAs(0.65)  // render at specific percent without changing state
```

---

## Viewport

```go
import "github.com/charmbracelet/bubbles/viewport"

vp := viewport.New(width, height)
vp.SetContent(longContent)

// Keyboard navigation built-in (pgup/pgdown/arrows)
// In Update:
m.viewport, cmd = m.viewport.Update(msg)

// Programmatic scroll
m.viewport.GotoTop()
m.viewport.GotoBottom()
m.viewport.ScrollDown(n)

// Percent scrolled
percent := m.viewport.ScrollPercent()

// In View:
m.viewport.View()
```

---

## Paginator

```go
import "github.com/charmbracelet/bubbles/paginator"

p := paginator.New()
p.Type = paginator.Dots       // Dots or Arabic
p.PerPage = 5
p.SetTotalPages(len(items))

// In Update:
m.paginator, cmd = m.paginator.Update(msg)

// Get current page items
start, end := m.paginator.GetSliceBounds(len(items))
visibleItems := items[start:end]

// In View:
fmt.Println(m.paginator.View())   // renders dots or "1/5"
```

---

## FilePicker

```go
import "github.com/charmbracelet/bubbles/filepicker"

fp := filepicker.New()
fp.AllowedTypes = []string{".go", ".txt"}
fp.CurrentDirectory, _ = os.UserHomeDir()

// In Init:
return fp.Init()

// In Update:
m.filepicker, cmd = m.filepicker.Update(msg)
if didSelect, path := m.filepicker.DidSelectFile(msg); didSelect {
    m.selectedFile = path
}
if didSelect, path := m.filepicker.DidSelectDisabledFile(msg); didSelect {
    // user tried to pick a disallowed file type
}
```

---

## Timer & Stopwatch

```go
import (
    "github.com/charmbracelet/bubbles/timer"
    "github.com/charmbracelet/bubbles/stopwatch"
)

// Timer (countdown)
t := timer.NewWithInterval(5*time.Minute, time.Second)
// In Init: return t.Init()
// In Update:
case timer.TickMsg:
    m.timer, cmd = m.timer.Update(msg)
case timer.TimeoutMsg:
    // timer expired

// Stopwatch (count up)
sw := stopwatch.NewWithInterval(time.Millisecond * 100)
// In Init: return sw.Init()
// In Update: m.stopwatch, cmd = m.stopwatch.Update(msg)
```

---

## Help

```go
import "github.com/charmbracelet/bubbles/help"
import "github.com/charmbracelet/bubbles/key"

// Define keybindings
type keyMap struct {
    Up   key.Binding
    Down key.Binding
    Quit key.Binding
}

var keys = keyMap{
    Up:   key.NewBinding(key.WithKeys("k", "up"), key.WithHelp("↑/k", "move up")),
    Down: key.NewBinding(key.WithKeys("j", "down"), key.WithHelp("↓/j", "move down")),
    Quit: key.NewBinding(key.WithKeys("q", "ctrl+c"), key.WithHelp("q", "quit")),
}

// keyMap must implement help.KeyMap interface
func (k keyMap) ShortHelp() []key.Binding { return []key.Binding{k.Up, k.Down, k.Quit} }
func (k keyMap) FullHelp() [][]key.Binding {
    return [][]key.Binding{{k.Up, k.Down}, {k.Quit}}
}

type model struct {
    help help.Model
    keys keyMap
}

func NewModel() model {
    return model{help: help.New(), keys: keys}
}

// In Update:
case tea.WindowSizeMsg:
    m.help.Width = msg.Width

case tea.KeyPressMsg:
    switch {
    case key.Matches(msg, m.keys.Up):   m.cursor--
    case key.Matches(msg, m.keys.Down): m.cursor++
    case key.Matches(msg, m.keys.Quit): return m, tea.Quit
    }

// In View:
m.help.View(m.keys)     // renders short help line
// User can toggle full/short help with '?'
```

---

## Key Bindings (standalone)

```go
import "github.com/charmbracelet/bubbles/key"

binding := key.NewBinding(
    key.WithKeys("j", "down"),         // actual keys
    key.WithHelp("↓/j", "move down"), // help text
)

// Match in Update
case tea.KeyPressMsg:
    if key.Matches(msg, binding) {
        // handle it
    }

// Disable a binding
binding.SetEnabled(false)
```
