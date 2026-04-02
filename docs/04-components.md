# Phase 4 — Component tree + stream compiler

## Step 11 — StreamOp types + compiler (`components/component.go`)

```go
package components

type StreamOp interface{ streamOp() }

// Graphics state
type SaveState    struct{}
type RestoreState struct{}

// Color
type SetFillColor   struct{ R, G, B float64 }
type SetStrokeColor struct{ R, G, B float64 }
type SetLineWidth   struct{ Width float64 }

// Paths
type Rectangle struct{ X, Y, W, H float64 }
type Fill        struct{}
type StrokePath  struct{}
type FillAndStroke struct{}

// Transform
type Transform struct{ A, B, C, D, E, F float64 }

// Text
type BeginText struct{}
type EndText   struct{}
type SetFont   struct{ Name string; Size float64 }
type MoveText  struct{ X, Y float64 }
type ShowText  struct{ Text string }

// XObject (images, forms)
type DrawXObject struct{ Name string }

// marker methods omitted for brevity — all implement StreamOp

// Compiler
func CompileOpsToBytes(ops []StreamOp) []byte
// produces valid PDF content stream bytes
// e.g. SaveState{} → "q\n", SetFillColor{0.1,0.4,0.8} → "0.1 0.4 0.8 rg\n"
```

### Test checklist

- [ ] CompileOpsToBytes([]StreamOp{SaveState{}, SetFillColor{1,0,0}, Rectangle{0,0,100,100}, Fill{}, RestoreState{}})
  matches `q\n1 0 0 rg\n0 0 100 100 re\nf\nQ\n`
- [ ] ShowText escapes parentheses in content string
- [ ] Transform produces correct `cm` operator

---

## Step 12 — Box, Text, Image (`components/`)

### Component interface

```go
type Component interface {
    ComponentID() string
    Render() []StreamOp
    Children() []Component
}
```

### Box (`components/box.go`)

```go
type Box struct {
    id       string
    X, Y, W, H float64
    Fill     *color.RGB
    Stroke   *color.RGB
    Border   float64
    children []Component
}

func NewBox(id string) *Box

// fluent setters — all return *Box
func (b *Box) WithBounds(x, y, w, h float64) *Box
func (b *Box) WithFill(c color.RGB) *Box
func (b *Box) WithStroke(c color.RGB, width float64) *Box
func (b *Box) AddChild(c Component) *Box

func (b *Box) ComponentID() string
func (b *Box) Children() []Component
func (b *Box) Render() []StreamOp
// SaveState → color ops → Rectangle → fill/stroke → children.Render() → RestoreState
```

### Text (`components/text.go`)

```go
type Text struct {
    id       string
    X, Y     float64
    Content  string
    FontName string
    FontSize float64
    Color    *color.RGB
}

func NewText(id string) *Text

func (t *Text) WithPosition(x, y float64) *Text
func (t *Text) WithFont(name string, size float64) *Text
func (t *Text) WithColor(c color.RGB) *Text
func (t *Text) WithContent(s string) *Text

func (t *Text) ComponentID() string
func (t *Text) Children() []Component  // always nil
func (t *Text) Render() []StreamOp
```

### Image (`components/image.go`)

```go
type Image struct {
    id       string
    X, Y, W, H float64
    XObjName string  // resource name e.g. "/Im1"
}

func NewImage(id, xObjName string) *Image
func (i *Image) WithBounds(x, y, w, h float64) *Image
func (i *Image) Render() []StreamOp
// SaveState → Transform (scale+translate) → DrawXObject → RestoreState
```

### Test checklist

- [ ] Box with no children: Render() produces q … re f Q sequence
- [ ] Box with Text child: child ops appear between parent fill and RestoreState
- [ ] Text.Render() produces BT … Tf … Td … Tj ET sequence
- [ ] Image.Render() produces correct cm matrix before Do operator

---

## Step 13 — Page component tree (`domain/nodes/page.go`)

```go
func (p *Page) AddComponent(c components.Component) *Page
func (p *Page) RemoveComponent(id string) *Page
func (p *Page) FindComponent(id string) components.Component  // recursive search

func (p *Page) Recompile() error
// 1. collect Render() from all top-level components
// 2. CompileOpsToBytes()
// 3. build new PdfStream with result
// 4. doc.ModifyObject(p.Contents.ID(), newStream)
```

### Test checklist

- [ ] AddComponent + Recompile: ObjectModified event pushed with correct stream bytes
- [ ] RemoveComponent: component gone from slice, Recompile reflects removal
- [ ] FindComponent: finds nested child by id
