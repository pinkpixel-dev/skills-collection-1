# Harmonica Reference

**Repo:** https://github.com/charmbracelet/harmonica  
**Docs:** https://pkg.go.dev/github.com/charmbracelet/harmonica  
**Import:** `github.com/charmbracelet/harmonica`

A simple, physics-based spring animation library. Framework-agnostic — works great with BubbleTea but doesn't depend on it.

---

## Core Concept

Harmonica simulates a **damped spring**. You give it:
1. A target value
2. The current value and velocity
3. It computes the next position and velocity

Call `Update()` every frame to animate toward the target.

---

## Basic Usage

```go
import "github.com/charmbracelet/harmonica"

// Initialize spring: framerate, angular frequency (speed), damping ratio
spring := harmonica.NewSpring(harmonica.FPS(60), 6.0, 0.5)

// State: current position + velocity
var (
    pos, vel float64
)
const target = 100.0

// Each frame:
pos, vel = spring.Update(pos, vel, target)
```

---

## Parameters

```go
// FPS helper converts framerate to time delta
harmonica.FPS(60)   // 60fps
harmonica.FPS(30)   // 30fps

// Or provide time delta directly (e.g., from a game engine)
spring := harmonica.NewSpring(1.0/60.0, 6.0, 0.5)
```

**Angular Frequency** (second param): speed. Higher = faster. ~5.0 is a good starting point.

**Damping Ratio** (third param): springiness.
- `< 1.0` = **under-damped**: bouncy, overshoots (e.g., 0.5 = playful)
- `= 1.0` = **critically-damped**: fastest without bounce
- `> 1.0` = **over-damped**: slow, no oscillation (sluggish)

---

## 2D Animation

Animate X and Y independently with the same spring:

```go
spring := harmonica.NewSpring(harmonica.FPS(60), 6.0, 0.5)

var (
    x, xVel float64
    y, yVel float64
)

const targetX, targetY = 50.0, 100.0

// Each frame:
x, xVel = spring.Update(x, xVel, targetX)
y, yVel = spring.Update(y, yVel, targetY)
```

---

## Integration with BubbleTea

```go
import (
    tea "charm.land/bubbletea/v2"
    "github.com/charmbracelet/harmonica"
    "time"
)

type model struct {
    spring    harmonica.Spring
    pos       float64
    vel       float64
    target    float64
    lastTick  time.Time
}

type tickMsg time.Time

func (m model) Init() tea.Cmd {
    return tea.Tick(time.Second/60, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tickMsg:
        m.pos, m.vel = m.spring.Update(m.pos, m.vel, m.target)
        return m, tea.Tick(time.Second/60, func(t time.Time) tea.Msg {
            return tickMsg(t)
        })
    }
    return m, nil
}
```

**Tip:** Bubbles' `progress` component uses Harmonica internally for smooth bar animation. You may not need Harmonica directly if using that component.

---

## Projectile Motion

Harmonica also includes a `Projectile` type for gravity-based motion:

```go
p := harmonica.NewProjectile(harmonica.FPS(60), harmonica.Point{X: 0, Y: 0}, harmonica.Point{X: 10, Y: -5}, harmonica.Point{X: 0, Y: 9.8})
// (framerate, initial position, initial velocity, gravity)

// Each frame:
p.Update()
x, y := p.Position()
```

---

---

# BubbleZone Reference

**Repo:** https://github.com/lrstanley/bubblezone  
**Docs:** https://pkg.go.dev/github.com/lrstanley/bubblezone/v2  
**Import:** `github.com/lrstanley/bubblezone/v2`  
**Version:** v2.x

Mouse event tracking for BubbleTea components. Allows you to define named "zones" and check if a mouse event landed inside them — no manual coordinate math needed.

> ⚠️ **Note:** BubbleZone v2 may have limitations with Lip Gloss v2's new compositor/canvas. For simple layouts it works great; for complex layered UIs, consider if native BubbleTea v2 mouse handling fits better.

---

## How It Works

1. **Initialize** the zone manager
2. **Mark** regions in your `View()` with `zone.Mark(id, content)`
3. **Scan** the root view output with `zone.Scan()`
4. **Check** mouse events with `zone.Get(id).InBounds(mouseMsg)`

---

## Setup

```go
import zone "github.com/lrstanley/bubblezone/v2"

func main() {
    zone.NewGlobal()
    // defer zone.Close() // if app continues after TUI closes
    
    p := tea.NewProgram(model{})
    p.Run()
}
```

---

## Mark Zones in View

```go
func (m model) View() tea.View {
    var view tea.View
    view.AltScreen = true  // REQUIRED — BubbleZone needs alt-screen
    view.MouseMode = tea.MouseModeCellMotion  // enable mouse
    
    // Wrap view content in zone.Scan at the ROOT model only
    content := lipgloss.JoinHorizontal(
        lipgloss.Top,
        zone.Mark("ok-btn",     m.renderOKButton()),
        zone.Mark("cancel-btn", m.renderCancelButton()),
    )
    view.SetContent(zone.Scan(content))
    return view
}

// In child models — just mark, no Scan
func (m childModel) View() string {
    return zone.Mark("my-component", m.renderContent())
}
```

---

## Check Bounds in Update

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.MouseReleaseMsg:
        if msg.Button != tea.MouseLeft {
            return m, nil
        }
        
        if zone.Get("ok-btn").InBounds(msg) {
            m.confirmed = true
        } else if zone.Get("cancel-btn").InBounds(msg) {
            return m, tea.Quit
        }
    
    case tea.MouseMotionMsg:
        // Check hover
        m.hovering = zone.Get("ok-btn").InBounds(msg)
    }
    return m, nil
}
```

---

## Get Relative Position

```go
// x, y relative to the zone (not screen)
x, y := zone.Get("my-component").Pos(msg.X, msg.Y)
```

---

## Unique IDs with Prefix

Use `NewPrefix()` to avoid ID collisions when multiple instances of a component exist:

```go
type listItem struct {
    id    string  // unique per item
    title string
}

func newItem(title string) listItem {
    return listItem{id: zone.NewPrefix(), title: title}
}

// In View:
func (m listItem) View() string {
    return zone.Mark(m.id, m.title)
}

// In Update:
if zone.Get(m.selectedItem.id).InBounds(msg) {
    // handle click on this specific item
}
```

---

## Tips

- **Only** call `zone.Scan()` in the **root model's View()** — not in child components
- Use `lipgloss.Width()` not `len()` — zone markers are zero-width to `lipgloss.Width()`
- Avoid `MaxHeight()`/`MaxWidth()` that could trim zone markers out of the output
- For organic/circular shapes, bounds checking is rectangular — pad appropriately

---

---

# ntcharts Reference

**Repo:** https://github.com/NimbleMarkets/ntcharts  
**Docs:** https://pkg.go.dev/github.com/NimbleMarkets/ntcharts  
**Import:** `github.com/NimbleMarkets/ntcharts` (each chart has its own sub-package)

Terminal charting library built for BubbleTea + Lip Gloss + BubbleZone.

---

## Chart Types

| Package | Type | Import |
|---------|------|--------|
| `canvas` | 2D grid base | `ntcharts/canvas` |
| `barchart` | Bar chart | `ntcharts/barchart` |
| `linechart` | Line chart family | `ntcharts/linechart` |
| `linechart/streamlinechart` | Streaming line | `ntcharts/linechart/streamlinechart` |
| `linechart/timeserieslinechart` | Time series | `ntcharts/linechart/timeserieslinechart` |
| `linechart/wavelinechart` | Wave line | `ntcharts/linechart/wavelinechart` |
| `sparkline` | Mini sparkline | `ntcharts/sparkline` |
| `heatmap` | Color heatmap | `ntcharts/heatmap` |

---

## Install

```bash
go get github.com/NimbleMarkets/ntcharts
```

---

## Usage Pattern (all charts)

```go
// 1. Create chart
c := <package>.New(width, height, options...)

// 2. Push data
c.Push(dataPoint)    // or PushAll([]dataPoint)

// 3. Draw (rasterize) — call EVERY frame before View
c.Draw()             // or DrawBraille() for braille rune style

// 4. Get string output
output := c.View()
```

---

## Canvas

Foundation for all charts. A 2D grid of runes with Lip Gloss styling.

```go
import "github.com/NimbleMarkets/ntcharts/canvas"

c := canvas.New(5, 2)
c.SetLinesWithStyle(
    []string{"hello", "world"},
    lipgloss.NewStyle().Foreground(lipgloss.Color("6")),
)
fmt.Println(c.View())
```

---

## Bar Chart

```go
import "github.com/NimbleMarkets/ntcharts/barchart"

d1 := barchart.BarData{
    Label: "Q1",
    Values: []barchart.BarValue{
        {"Revenue", 42.5, lipgloss.NewStyle().Foreground(lipgloss.Color("10"))},
        {"Expenses", 30.1, lipgloss.NewStyle().Foreground(lipgloss.Color("9"))},
    },
}

bc := barchart.New(20, 10)
bc.PushAll([]barchart.BarData{d1, d2, d3})
bc.Draw()
fmt.Println(bc.View())

// Horizontal bars
bc := barchart.New(30, 10, barchart.WithHorizontalBars())
```

---

## Sparkline

```go
import "github.com/NimbleMarkets/ntcharts/sparkline"

sl := sparkline.New(10, 5)
sl.PushAll([]float64{7.81, 3.82, 8.39, 2.06, 4.19, 4.34, 6.83, 2.51, 9.21, 1.3})
sl.Draw()
fmt.Println(sl.View())

// Style
sl := sparkline.New(20, 4,
    sparkline.WithStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("82"))),
)
```

---

## Streamline Chart (scrolling line)

```go
import "github.com/NimbleMarkets/ntcharts/linechart/streamlinechart"

slc := streamlinechart.New(40, 10)
// Push values — new data scrolls in from the right
for _, v := range []float64{4, 6, 8, 10, 8, 6, 4} {
    slc.Push(v)
}
slc.Draw()
fmt.Println(slc.View())

// With style
slc := streamlinechart.New(40, 10,
    streamlinechart.WithLineStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("208"))),
    streamlinechart.WithXYSteps(2, 2),  // axis label step size
)
```

---

## Time Series Chart

```go
import "github.com/NimbleMarkets/ntcharts/linechart/timeserieslinechart"

tslc := timeserieslinechart.New(60, 15)

for i, v := range []float64{0, 4, 8, 10, 8, 4, 0, -4, -8} {
    date := time.Now().Add(time.Hour * 24 * time.Duration(i))
    tslc.Push(timeserieslinechart.TimePoint{Time: date, Value: v})
}

tslc.Draw()         // block character rendering
// or
tslc.DrawBraille()  // braille rendering (more detail)

fmt.Println(tslc.View())
```

---

## Waveline Chart

```go
import (
    "github.com/NimbleMarkets/ntcharts/canvas"
    "github.com/NimbleMarkets/ntcharts/linechart/wavelinechart"
)

wlc := wavelinechart.New(20, 10,
    wavelinechart.WithYRange(-3, 3),
)

// Plot individual points
wlc.Plot(canvas.Float64Point{X: 1.0, Y: 2.0})
wlc.Plot(canvas.Float64Point{X: 3.0, Y: -2.0})
wlc.Plot(canvas.Float64Point{X: 5.0, Y: 2.0})

wlc.Draw()
fmt.Println(wlc.View())
```

---

## Heatmap

```go
import "github.com/NimbleMarkets/ntcharts/heatmap"

hm := heatmap.New(20, 20, heatmap.WithValueRange(0, 1))
hm.SetXYRange(-1, 1, -1, 1)

for x := -1.0; x < 1.0; x += 0.05 {
    for y := -1.0; y < 1.0; y += 0.05 {
        val := math.Sin(math.Sqrt(x*x + y*y))
        hm.Push(heatmap.NewHeatPoint(x, y, val))
    }
}
hm.Draw()
fmt.Println(hm.View())
```

---

## Embedding in BubbleTea

Charts are not `tea.Model` themselves — embed in your model and call Draw/View:

```go
type model struct {
    chart streamlinechart.Model
    width int
    height int
}

func (m model) Init() tea.Cmd {
    return tea.Tick(100*time.Millisecond, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg.(type) {
    case tickMsg:
        // Push new data each tick
        m.chart.Push(newDataPoint())
        return m, tea.Tick(100*time.Millisecond, func(t time.Time) tea.Msg {
            return tickMsg(t)
        })
    case tea.WindowSizeMsg:
        // Resize chart (create new one with new dimensions)
        m.chart = streamlinechart.New(msg.Width-4, msg.Height-4)
    }
    return m, nil
}

func (m model) View() tea.View {
    m.chart.Draw()  // MUST call Draw before View every frame
    return tea.NewView(
        lipgloss.NewStyle().Border(lipgloss.RoundedBorder()).Render(m.chart.View()),
    )
}
```

---

## With BubbleZone (Mouse Support)

ntcharts' Canvas supports BubbleZone for mouse interaction:

```go
import zone "github.com/lrstanley/bubblezone/v2"

// Charts created with zone support
slc := streamlinechart.New(40, 10,
    streamlinechart.WithZoneManager(zone.DefaultManager()),
)

// Mark the chart zone
content := zone.Mark("my-chart", slc.View())
```

---

## Chart Style Options (common patterns)

```go
// Most charts accept functional options
chart := streamlinechart.New(width, height,
    streamlinechart.WithLineStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("82"))),
    streamlinechart.WithAxesStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("240"))),
    streamlinechart.WithXYRange(0, 100, 0, 50),
    streamlinechart.WithXYSteps(5, 10),  // axis label frequency
)
```

See the [examples README](https://github.com/NimbleMarkets/ntcharts/blob/main/examples/README.md) for full option lists per chart type.
