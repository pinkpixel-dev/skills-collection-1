# Huh Reference

**Repo:** https://github.com/charmbracelet/huh  
**Docs:** https://pkg.go.dev/charm.land/huh/v2  
**Import:** `charm.land/huh/v2`  
**Version:** v2.x

Simple, powerful library for building terminal forms and prompts. Works standalone OR embedded in BubbleTea.

---

## Form Structure

```
Form → Groups → Fields
```

- **Form**: The top-level container
- **Group**: A "page" of fields (shown together on screen)
- **Fields**: `Input`, `Text`, `Select`, `MultiSelect`, `Confirm`, `FilePicker`, `Note`

---

## Quick Start

```go
import "charm.land/huh/v2"

var (
    name     string
    confirm  bool
    choice   string
)

form := huh.NewForm(
    huh.NewGroup(
        huh.NewInput().Title("What's your name?").Value(&name),
        huh.NewSelect[string]().
            Title("Favorite color?").
            Options(huh.NewOptions("Red", "Blue", "Green")...).
            Value(&choice),
        huh.NewConfirm().Title("Confirm?").Value(&confirm),
    ),
)

if err := form.Run(); err != nil {
    log.Fatal(err)
}
fmt.Printf("Name: %s, Choice: %s, Confirmed: %v\n", name, choice, confirm)
```

---

## Fields

### Input (single line)

```go
var name string

huh.NewInput().
    Title("Name").
    Placeholder("Enter your name...").
    Prompt("? ").
    CharLimit(100).
    Value(&name).
    Validate(func(s string) error {
        if s == "" { return errors.New("name is required") }
        return nil
    })

// Standalone (blocking)
huh.NewInput().Title("Name?").Value(&name).Run()
```

### Text (multi-line)

```go
var story string

huh.NewText().
    Title("Tell me a story").
    Placeholder("Once upon a time...").
    CharLimit(500).
    Value(&story)
```

### Select (single choice)

```go
var color string

huh.NewSelect[string]().
    Title("Pick a color").
    Options(
        huh.NewOption("Red",   "red"),
        huh.NewOption("Green", "green"),
        huh.NewOption("Blue",  "blue"),
    ).
    Value(&color)

// From slice shorthand
huh.NewSelect[string]().Options(huh.NewOptions("A", "B", "C")...).Value(&choice)
```

### MultiSelect (multiple choices)

```go
var toppings []string

huh.NewMultiSelect[string]().
    Title("Toppings").
    Options(
        huh.NewOption("Cheese",  "cheese").Selected(true),
        huh.NewOption("Lettuce", "lettuce"),
        huh.NewOption("Bacon",   "bacon"),
    ).
    Limit(3).  // max selections
    Value(&toppings)
```

### Confirm (yes/no)

```go
var yes bool

huh.NewConfirm().
    Title("Are you sure?").
    Affirmative("Yes!").
    Negative("No.").
    Value(&yes)
```

### FilePicker

```go
var path string

huh.NewFilePicker().
    Title("Pick a file").
    CurrentDirectory("/home/user").
    AllowedTypes([]string{".go", ".txt"}).
    Value(&path)
```

### Note (display-only)

```go
huh.NewNote().
    Title("Important").
    Description("This is an important notice.\nPlease read carefully.").
    Next(true)  // show next button
```

---

## Validation

Every input field supports `.Validate(func(val T) error)`:

```go
huh.NewInput().
    Validate(func(s string) error {
        if len(s) < 3 { return fmt.Errorf("must be at least 3 chars") }
        return nil
    })
```

The form will display an error and prevent progression until validation passes.

---

## Themes

```go
form := huh.NewForm(...).WithTheme(huh.ThemeCharm())  // Default themes:
// huh.ThemeCharm()
// huh.ThemeDracula()
// huh.ThemeCatppuccin()
// huh.ThemeBase16()
// huh.ThemeDefault()
```

Custom theme via Lip Gloss:
```go
t := huh.ThemeCharm()
t.Focused.Title = lipgloss.NewStyle().Foreground(lipgloss.Color("#FF75B7"))
form.WithTheme(t)
```

---

## Accessibility Mode

```go
accessible := os.Getenv("ACCESSIBLE") != ""
form.WithAccessible(accessible)
// Drops TUI in favor of standard prompts for screen readers
```

---

## Dynamic Forms

Fields can recompute their properties based on other fields:

```go
var country, state string

huh.NewSelect[string]().
    Options(huh.NewOptions("United States", "Canada", "Mexico")...).
    Value(&country).
    Title("Country"),

huh.NewSelect[string]().
    Value(&state).
    TitleFunc(func() string {
        switch country {
        case "United States": return "State"
        case "Canada":        return "Province"
        default:              return "Territory"
        }
    }, &country).  // binding = recompute when country changes
    OptionsFunc(func() []huh.Option[string] {
        // huh handles caching — won't call API on every keystroke
        return fetchStatesFor(country)
    }, &country),
```

---

## Multi-Group Forms (Pages)

```go
form := huh.NewForm(
    huh.NewGroup(/* page 1 fields */),
    huh.NewGroup(/* page 2 fields */),
    huh.NewGroup(/* page 3 fields */).WithHideFunc(func() bool {
        return !showPage3  // conditionally hide a group
    }),
)
```

---

## Integration with BubbleTea

`huh.Form` is a `tea.Model`. Embed it in your model:

```go
type Model struct {
    form *huh.Form
}

func NewModel() Model {
    return Model{
        form: huh.NewForm(
            huh.NewGroup(
                huh.NewSelect[string]().
                    Key("action").
                    Options(huh.NewOptions("Create", "Update", "Delete")...).
                    Title("Action"),
            ),
        ),
    }
}

func (m Model) Init() tea.Cmd { return m.form.Init() }

func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    form, cmd := m.form.Update(msg)
    if f, ok := form.(*huh.Form); ok {
        m.form = f
    }
    // Check completion
    if m.form.State == huh.StateCompleted {
        action := m.form.GetString("action")
        _ = action // use it
        return m, tea.Quit
    }
    return m, cmd
}

func (m Model) View() string {
    if m.form.State == huh.StateCompleted {
        return "Done! Selected: " + m.form.GetString("action")
    }
    return m.form.View()
}
```

### Getting Values (KeyMap style)

When using `.Key("fieldname")` on fields, retrieve values from the completed form:

```go
m.form.GetString("fieldname")    // string fields
m.form.GetInt("fieldname")       // int fields  
m.form.GetBool("fieldname")      // confirm fields
m.form.GetStringSlice("fieldname") // multiselect fields
```

---

## Spinner (huh subpackage)

```go
import "charm.land/huh/v2/spinner"

// Action-based (blocking, runs action in goroutine)
err := spinner.New().
    Title("Processing...").
    Type(spinner.Line).  // Line, Dots, Meter, MiniDot, Jump, Pulse, Points, Globe, Moon
    Action(func() {
        time.Sleep(2 * time.Second)  // your long-running task
    }).
    Run()

// Context-based (non-blocking action)
ctx, cancel := context.WithCancel(context.Background())
go func() {
    doWork()
    cancel()
}()

err := spinner.New().
    Title("Working...").
    Context(ctx).
    Run()
```

---

## Form Options

```go
form := huh.NewForm(...).
    WithWidth(80).                      // set width
    WithHeight(20).                     // set height
    WithShowHelp(false).               // hide help bar
    WithShowErrors(true).              // show errors (default)
    WithKeyMap(customKeyMap)           // custom keybindings
```

---

## Known Gotcha

Huh v2 requires BubbleTea v2. Make sure your `go.mod` has matching v2 versions for both.
