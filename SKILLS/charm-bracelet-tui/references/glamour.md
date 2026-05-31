# Glamour Reference

**Repo:** https://github.com/charmbracelet/glamour  
**Docs:** https://pkg.go.dev/github.com/charmbracelet/glamour  
**Import:** `github.com/charmbracelet/glamour`

Stylesheet-based Markdown renderer for terminals. Renders Markdown with syntax highlighting, styled headers, code blocks, lists, tables, and more.

---

## Quick Start

```go
import "github.com/charmbracelet/glamour"

md := `# Hello World

This is **bold** and _italic_ markdown.

` + "```go\nfmt.Println(\"Hello!\")\n```"

// Simple render with built-in style
out, err := glamour.Render(md, "dark")
if err != nil { log.Fatal(err) }
fmt.Print(out)
```

---

## Built-In Styles

| Style | Description |
|-------|-------------|
| `"dark"` | Dark background theme |
| `"light"` | Light background theme |
| `"auto"` | Auto-detects terminal background |
| `"notty"` | Plain text, no ANSI (for non-TTY output) |
| `"ascii"` | ASCII-only, no Unicode |
| `"dracula"` | Dracula color scheme |

```go
out, err := glamour.Render(md, "dark")
out, err := glamour.Render(md, "auto")    // auto-detect dark/light
out, err := glamour.Render(md, "notty")  // no colors (for pipes/logs)
```

---

## Custom Renderer

```go
r, err := glamour.NewTermRenderer(
    glamour.WithAutoStyle(),           // detect dark/light background
    glamour.WithWordWrap(80),          // word wrap at 80 chars (default)
    glamour.WithWordWrap(0),           // disable word wrap
    glamour.WithStyles(myStyleConfig), // custom style object
    glamour.WithEnvironmentConfig(),   // use GLAMOUR_STYLE env var
    glamour.WithEmoji(),               // enable emoji rendering
    glamour.WithPreservedNewLines(),   // preserve blank lines
)
if err != nil { log.Fatal(err) }
defer r.Close()

out, err := r.Render(md)
fmt.Print(out)
```

---

## Environment Variable Config

Set `GLAMOUR_STYLE` to a style name or path to a JSON style file:

```bash
GLAMOUR_STYLE=dark ./myapp
GLAMOUR_STYLE=/path/to/custom-style.json ./myapp
```

```go
// Render using GLAMOUR_STYLE env var
out, err := glamour.RenderWithEnvironmentConfig(md)

// Or pass it to a renderer
r, _ := glamour.NewTermRenderer(glamour.WithEnvironmentConfig())
```

---

## Custom Styles (JSON)

Glamour styles are JSON objects. Create a `StyleConfig` or use a JSON file. Style keys correspond to Markdown elements.

```go
import "github.com/charmbracelet/glamour/ansi"

styleConfig := ansi.StyleConfig{
    Document: ansi.StyleBlock{
        StylePrimitive: ansi.StylePrimitive{
            BlockPrefix:  "\n",
            BlockSuffix:  "\n",
            Color:        stringPtr("248"),
        },
        Margin: uintPtr(2),
    },
    Heading: ansi.StyleBlock{
        StylePrimitive: ansi.StylePrimitive{
            BlockSuffix: "\n",
            Color:       stringPtr("39"),
            Bold:        boolPtr(true),
        },
    },
    H1: ansi.StyleBlock{
        StylePrimitive: ansi.StylePrimitive{
            Prefix: "# ",
            Color:  stringPtr("81"),
            Bold:   boolPtr(true),
        },
    },
    // H2, H3, H4, H5, H6 similar...
    CodeBlock: ansi.StyleCodeBlock{
        StyleBlock: ansi.StyleBlock{
            StylePrimitive: ansi.StylePrimitive{
                Color: stringPtr("204"),
            },
            Margin: uintPtr(2),
        },
        Chroma: &ansi.Chroma{ /* syntax highlighter config */ },
    },
    Link: ansi.StylePrimitive{
        Color:     stringPtr("30"),
        Underline: boolPtr(true),
    },
    // Strong, Emph, Strikethrough, BlockQuote, List, Item, Table, etc.
}

r, _ := glamour.NewTermRenderer(glamour.WithStyles(styleConfig))
```

Helper pointer functions (needed for the config struct):
```go
func stringPtr(s string) *string { return &s }
func boolPtr(b bool) *bool       { return &b }
func uintPtr(u uint) *uint       { return &u }
```

---

## Style Gallery

All default styles are in: https://github.com/charmbracelet/glamour/tree/master/styles/gallery

---

## Using in BubbleTea

Glamour renders markdown to a string, then display with a viewport:

```go
import (
    "github.com/charmbracelet/bubbles/viewport"
    "github.com/charmbracelet/glamour"
)

type model struct {
    viewport viewport.Model
    ready    bool
}

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        r, _ := glamour.NewTermRenderer(
            glamour.WithAutoStyle(),
            glamour.WithWordWrap(msg.Width),
        )
        rendered, _ := r.Render(markdownContent)
        
        if !m.ready {
            m.viewport = viewport.New(msg.Width, msg.Height)
            m.ready = true
        } else {
            m.viewport.Width = msg.Width
            m.viewport.Height = msg.Height
        }
        m.viewport.SetContent(rendered)
    }
    m.viewport, cmd = m.viewport.Update(msg)
    return m, cmd
}

func (m model) View() string {
    if !m.ready { return "Loading..." }
    return m.viewport.View()
}
```

---

## What Glamour Renders

Glamour handles all standard Markdown:
- Headers (H1–H6) with styled colors
- Bold, italic, strikethrough
- Code blocks with syntax highlighting (via Chroma)
- Inline code
- Links (clickable in supporting terminals)
- Blockquotes
- Ordered and unordered lists
- Tables
- Horizontal rules
- Images (shows alt text)
- HTML (stripped or shown as-is depending on config)

---

## Notable Projects Using Glamour

- **Glow** — a terminal markdown reader (by Charm)
- **GitHub CLI** (`gh`) — renders PR/issue descriptions
- **GitLab CLI** — renders issue/MR descriptions
