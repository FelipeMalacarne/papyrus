# Phase 2 — Document aggregate + lazy resolver

## Step 4 — Document + ObjectResolver (`domain/document.go`)

The aggregate root. Never does I/O directly — delegates to an injected ObjectResolver.

```go
package domain

type ObjectResolver interface {
    ReadObjectAt(offset int64) (PdfObject, error)
}

type Document struct {
    xref     *XRefTable
    cache    map[uint32]PdfObject
    events   []DomainEvent
    resolver ObjectResolver   // nil for new documents

    RootID ObjectID
    InfoID ObjectID
}

func NewDocument(xref *XRefTable, resolver ObjectResolver) *Document

// Resolve: cache hit → return; cache miss → xref lookup → resolver.ReadObjectAt()
func (d *Document) Resolve(id ObjectID) (PdfObject, error)

func (d *Document) ModifyObject(id ObjectID, obj PdfObject)
func (d *Document) RemoveObject(id ObjectID)
func (d *Document) AllocateID() ObjectID

func (d *Document) PendingEvents() []DomainEvent
func (d *Document) ClearEvents()
```

### Test checklist

- [ ] Mock resolver: Resolve same id twice, assert ReadObjectAt called once
- [ ] ModifyObject updates cache and pushes ObjectModified event
- [ ] RemoveObject deletes from cache, marks xref free, pushes ObjectRemoved
- [ ] AllocateID returns incrementing ids across calls

---

## Step 5 — NodeRef[T] (`domain/noderef.go`)

Typed lazy handle for every edge in the PDF object graph.

```go
package domain

type PdfNode interface{ pdfNode() }

type NodeRef[T PdfNode] struct {
    id       ObjectID
    doc      *Document
    cached   T
    resolved bool
}

func Ref[T PdfNode](id ObjectID, doc *Document) *NodeRef[T]

// Get: returns cached value or resolves via doc.Resolve()
func (r *NodeRef[T]) Get() (T, error)

// Modify: resolves, applies fn, calls doc.ModifyObject()
func (r *NodeRef[T]) Modify(fn func(T) T) error

// Remove: calls doc.RemoveObject()
func (r *NodeRef[T]) Remove()

func (r *NodeRef[T]) ID() ObjectID
```

### Test checklist

- [ ] Get() on cold ref calls doc.Resolve once; second Get() returns cached
- [ ] Modify() pushes ObjectModified with updated value
- [ ] Remove() pushes ObjectRemoved and clears cache

---

## Step 6 — Node types (`domain/nodes/`)

### Catalog (`catalog.go`)

```go
type Catalog struct {
    Pages *NodeRef[*PageTree]
    Info  *NodeRef[*InfoDict]
    id    ObjectID
    doc   *Document
}

func (c *Catalog) pdfNode() {}
func (c *Catalog) GetPages() (*PageTree, error)
```

### PageTree (`page_tree.go`)

```go
type PageTree struct {
    kids  []*NodeRef[*Page]
    count int
    id    ObjectID
    doc   *Document
}

func (pt *PageTree) pdfNode() {}
func (pt *PageTree) GetPage(n int) (*Page, error)
func (pt *PageTree) AddPage(page *Page) error
func (pt *PageTree) RemovePage(n int) error
func (pt *PageTree) Count() int
```

### Page (`page.go`)

```go
type Page struct {
    MediaBox  [4]float64
    Rotate    int
    Contents  *NodeRef[*ContentStream]
    Resources *NodeRef[*ResourceDict]
    id        ObjectID
    doc       *Document
    components []components.Component
}

func (p *Page) pdfNode() {}
func (p *Page) GetContents() (*ContentStream, error)
func (p *Page) GetResources() (*ResourceDict, error)
func (p *Page) AddComponent(c components.Component) *Page
func (p *Page) RemoveComponent(id string) *Page
func (p *Page) FindComponent(id string) components.Component
func (p *Page) Recompile() error   // re-renders component tree → ModifyObject on Contents
```

### Test checklist

- [ ] PageTree.GetPage(n) returns correct NodeRef id
- [ ] PageTree.AddPage pushes PageAdded event
- [ ] PageTree.RemovePage pushes PageRemoved event and removes kid
- [ ] Page.AddComponent + Recompile pushes ObjectModified on Contents id
