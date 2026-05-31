# Lip Gloss Reference

**Repo:** https://github.com/charmbracelet/lipgloss  
**Docs:** https://pkg.go.dev/charm.land/lipgloss/v2  
**Import:** `charm.land/lipgloss/v2`  
**Version:** v2.x

---

## Style Basics

```go
import "charm.land/lipgloss/v2"

// Create a style (pure value type — copy freely)
style := lipgloss.NewStyle().
    Bold(true).
    Italic(true).
    Foreground(lipgloss.Color("#FAFAFA")).
    Background(lipgloss.Color("#7D56F4")).
    PaddingTop(2).
    PaddingLeft(4).
    Width(22)

// Render
output := style.Render("Hello, world!")

// Or use Println (auto-downsamples colors)
lipgloss.Println(style.Render("Hello!"))
```

---

## Colors

```go
// True color (24-bit)
lipgloss.Color("#0000FF")
lipgloss.Color("#FF75B7")

// ANSI 256 (8-bit)
lipgloss.Color("86")   // aqua
lipgloss.Color("201")  // hot pink

// ANSI 16 (4-bit)
lipgloss.Color("5")    // magenta
lipgloss.Color("9")    // red

// Named constants
lipgloss.Black, lipgloss.Red, lipgloss.Green, lipgloss.Yellow
lipgloss.Blue, lipgloss.Magenta, lipgloss.Cyan, lipgloss.White
// Bright variants: lipgloss.BrightBlack, etc.

// Color utilities
dark    := lipgloss.Darken(c, 0.5)
light   := lipgloss.Lighten(c, 0.35)
comp    := lipgloss.Complementary(c)
alpha   := lipgloss.Alpha(c, 0.2)

// Adaptive colors (dark/light theme detection)
hasDark := lipgloss.HasDarkBackground(os.Stdin, os.Stdout)
ld      := lipgloss.LightDark(hasDark)
myColor := ld(lipgloss.Color("#D7FFAE"), lipgloss.Color("#D75FEE"))

// With BubbleTea v2 (preferred — no I/O race)
func (m model) Init() tea.Cmd { return tea.RequestBackgroundColor }
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    if bg, ok := msg.(tea.BackgroundColorMsg); ok {
        m.ld = lipgloss.LightDark(bg.IsDark())
    }
    // ...
}

// Color gradients for layouts
colors1D := lipgloss.Blend1D(10, lipgloss.Color("#FF0000"), lipgloss.Color("#0000FF"))
colors2D := lipgloss.Blend2D(80, 24, 45.0, c1, c2, c3)
```

---

## Text Formatting

```go
style := lipgloss.NewStyle().
    Bold(true).
    Italic(true).
    Faint(true).
    Strikethrough(true).
    Underline(true).
    Reverse(true).
    Blink(true)

// Underline styles
s := lipgloss.NewStyle().
    UnderlineStyle(lipgloss.UnderlineCurly).  // or Single, Double, Dotted, Dashed
    UnderlineColor(lipgloss.Color("#FF0000"))

// Hyperlinks (supported terminals)
s := lipgloss.NewStyle().
    Foreground(lipgloss.Color("#7B2FBE")).
    Hyperlink("https://charm.land")
```

---

## Block Layout

```go
// Padding (shorthand follows CSS pattern)
lipgloss.NewStyle().Padding(2)              // all sides
lipgloss.NewStyle().Padding(2, 4)          // vertical, horizontal
lipgloss.NewStyle().Padding(1, 4, 2)       // top, sides, bottom
lipgloss.NewStyle().Padding(2, 4, 3, 1)    // top, right, bottom, left

// Margins (same shorthand)
lipgloss.NewStyle().Margin(1, 2)

// Width and Height (minimum)
lipgloss.NewStyle().Width(24).Height(10)

// Maximum constraints
lipgloss.NewStyle().MaxWidth(80).MaxHeight(25)

// Inline (force single line)
lipgloss.NewStyle().Inline(true).MaxWidth(5).Render("yadda yadda")

// Text alignment
lipgloss.NewStyle().Width(24).Align(lipgloss.Center)  // Left, Right, Center

// Tab width
lipgloss.NewStyle().TabWidth(4)  // default; 0 removes tabs, NoTabConversion keeps raw
```

---

## Borders

```go
// Border styles: NormalBorder, RoundedBorder, ThickBorder, DoubleBorder,
//               HiddenBorder, ASCIIBorder, MarkdownBorder

style := lipgloss.NewStyle().
    BorderStyle(lipgloss.RoundedBorder()).
    BorderForeground(lipgloss.Color("228")).
    BorderBackground(lipgloss.Color("63")).
    BorderTop(true).
    BorderLeft(true)

// Shorthand: (style, top, right, bottom, left)
lipgloss.NewStyle().Border(lipgloss.ThickBorder(), true, false) // top+bottom only

// Gradient border
s := lipgloss.NewStyle().
    Border(lipgloss.RoundedBorder()).
    BorderForegroundBlend(lipgloss.Color("#FF0000"), lipgloss.Color("#0000FF"))

// Custom border
var myCuteBorder = lipgloss.Border{
    Top: "._.:*:", Bottom: "._.:*:", Left: "|*", Right: "|*",
    TopLeft: "*", TopRight: "*", BottomLeft: "*", BottomRight: "*",
}
```

---

## Compositing / Layout

### Joining

```go
// Horizontal join (align by position: Top, Center, Bottom, or 0.0–1.0)
out := lipgloss.JoinHorizontal(lipgloss.Bottom, panelA, panelB, panelC)

// Vertical join
out := lipgloss.JoinVertical(lipgloss.Center, panelA, panelB)
```

### Measuring

```go
w := lipgloss.Width(block)    // ALWAYS use this, not len()
h := lipgloss.Height(block)
w, h := lipgloss.Size(block)
```

### Placing in Whitespace

```go
// Center horizontally in 80-col space
block := lipgloss.PlaceHorizontal(80, lipgloss.Center, content)

// Place at bottom of 30-row space
block := lipgloss.PlaceVertical(30, lipgloss.Bottom, content)

// Place in a 80x30 space (bottom-right)
block := lipgloss.Place(80, 30, lipgloss.Right, lipgloss.Bottom, content)
```

### Layered Compositing (v2 new)

```go
// Create layers with position and Z-order
a := lipgloss.NewLayer(pickles).X(4).Y(2).Z(1)
b := lipgloss.NewLayer(bitterMelon).X(22).Y(1)
c := lipgloss.NewLayer(sriracha).X(11).Y(7)

// Compose and render
output := compositor.Compose(a, b, c).Render()
```

### Text Wrapping

```go
wrapped := lipgloss.Wrap(styledText, 40, " ")
```

---

## Tables

```go
import "charm.land/lipgloss/v2/table"

headerStyle  := lipgloss.NewStyle().Foreground(lipgloss.Color("99")).Bold(true)
cellStyle    := lipgloss.NewStyle().Padding(0, 1).Width(14)
oddRowStyle  := cellStyle.Foreground(lipgloss.Color("245"))
evenRowStyle := cellStyle.Foreground(lipgloss.Color("241"))

t := table.New().
    Border(lipgloss.NormalBorder()).
    BorderStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("99"))).
    StyleFunc(func(row, col int) lipgloss.Style {
        switch {
        case row == table.HeaderRow: return headerStyle
        case row%2 == 0:            return evenRowStyle
        default:                    return oddRowStyle
        }
    }).
    Headers("NAME", "EMAIL", "ROLE").
    Rows(
        []string{"Alice", "alice@example.com", "admin"},
        []string{"Bob",   "bob@example.com",   "user"},
    )

lipgloss.Println(t)

// Markdown or ASCII style
table.New().Border(lipgloss.MarkdownBorder()).BorderTop(false).BorderBottom(false)
table.New().Border(lipgloss.ASCIIBorder())
```

---

## Lists

```go
import "charm.land/lipgloss/v2/list"

// Simple list
l := list.New("A", "B", "C")
lipgloss.Println(l)  // • A  • B  • C

// Nested
l := list.New(
    "Fruits", list.New("Apple", "Banana", "Cherry"),
    "Veggies", list.New("Carrot", "Daikon"),
)

// Custom enumerator & styles
l := list.New("Item1", "Item2").
    Enumerator(list.Roman).                              // Arabic, Alphabet, Roman, Bullet, Tree
    EnumeratorStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("99")).MarginRight(1)).
    ItemStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("212")))

// Custom enumerator function
func MyEnum(items list.Items, i int) string { return "→" }
```

---

## Trees

```go
import "charm.land/lipgloss/v2/tree"

t := tree.Root(".").
    Child("A", "B", "C")

// Nested
t := tree.Root("project").
    Child("src", tree.New().Child("main.go", "utils.go")).
    Child("tests", tree.New().Child("main_test.go"))

// Custom styles
t := tree.Root("⁜ Root").
    Enumerator(tree.RoundedEnumerator).    // DefaultEnumerator or RoundedEnumerator
    EnumeratorStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("63"))).
    RootStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("35"))).
    ItemStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("212")))
```

---

## Style Inheritance & Unsetting

```go
base := lipgloss.NewStyle().Foreground(lipgloss.Color("229")).Background(lipgloss.Color("63"))

// Inherit (only unset rules are inherited)
derived := lipgloss.NewStyle().Foreground(lipgloss.Color("201")).Inherit(base)

// Unset rules
style := lipgloss.NewStyle().Bold(true).UnsetBold().Background(lipgloss.Color("227")).UnsetBackground()
```

---

## Print Functions (color-aware, downsample-safe)

```go
lipgloss.Print(...)    // fmt.Print equivalent
lipgloss.Println(...)  // fmt.Println equivalent
lipgloss.Printf(...)   // fmt.Printf equivalent
lipgloss.Fprint(os.Stderr, ...)   // write to specific writer
lipgloss.Sprint(...)   // return string with downsampled colors
lipgloss.Sprintf(...)
```

Use these instead of `fmt.Print*` to get automatic color downsampling for non-TTY outputs.

---

## v1 → v2 Migration

The `compat` package provides `AdaptiveColor` and `CompleteColor` for easier migration:

```go
import "charm.land/lipgloss/v2/compat"

color := compat.AdaptiveColor{
    Light: lipgloss.Color("#f1f1f1"),
    Dark:  lipgloss.Color("#cccccc"),
}
```

Full upgrade guide: https://github.com/charmbracelet/lipgloss/blob/main/UPGRADE_GUIDE_V2.md
