# Phase 1 — Object model + XRef

## Step 1 — PdfObject types (`domain/object.go`)

All PDF value types implement the `PdfObject` marker interface.

```go
package domain

type PdfObject interface{ pdfObject() }

type PdfDict   map[string]PdfObject
type PdfArray  []PdfObject
type PdfName   string
type PdfString string
type PdfInt    int64
type PdfReal   float64
type PdfBool   bool
type PdfNull   struct{}

type PdfRef struct {
    ID  uint32
    Gen uint16
}

type PdfStream struct {
    Dict PdfDict
    Data []byte // decoded
}

type ObjectID = PdfRef

func (PdfDict) pdfObject()   {}
func (PdfArray) pdfObject()  {}
func (PdfName) pdfObject()   {}
func (PdfString) pdfObject() {}
func (PdfInt) pdfObject()    {}
func (PdfReal) pdfObject()   {}
func (PdfBool) pdfObject()   {}
func (PdfNull) pdfObject()   {}
func (PdfRef) pdfObject()    {}
func (PdfStream) pdfObject() {}
```

### Test checklist

- [ ] type switch on each concrete type returns correct branch
- [ ] PdfDict key lookup returns correct PdfObject
- [ ] PdfRef equality by value (ID + Gen)

---

## Step 2 — XRefTable (`domain/xref.go`)

In-memory index of all objects in the file.

```go
package domain

type XRefEntry struct {
    ObjID ObjectID
    Offset int64
    InUse  bool
}

type XRefTable struct {
    entries  map[uint32]XRefEntry
    maxID    uint32
}

func NewXRefTable() *XRefTable {
    return &XRefTable{entries: make(map[uint32]XRefEntry)}
}

func (x *XRefTable) Lookup(id uint32) (XRefEntry, bool)
func (x *XRefTable) Insert(entry XRefEntry)
func (x *XRefTable) MarkFree(id ObjectID)
func (x *XRefTable) NextFreeID() uint32
func (x *XRefTable) All() []XRefEntry   // for full rewrite serializer
```

### Test checklist

- [ ] Insert 10 entries, Lookup each by id
- [ ] MarkFree removes from active entries
- [ ] NextFreeID returns max+1 when no free slots
- [ ] All() returns all InUse entries in id order

---

## Step 3 — DomainEvent types (`domain/events.go`)

Events emitted by Document mutations. Consumed by the serializer.

```go
package domain

type DomainEvent interface{ domainEvent() }

type ObjectModified struct {
    ObjID    ObjectID
    NewValue PdfObject
}

type ObjectRemoved struct {
    ObjID ObjectID
}

type PageAdded struct {
    PageNum int
}

type PageRemoved struct {
    PageNum int
}

func (ObjectModified) domainEvent() {}
func (ObjectRemoved) domainEvent()  {}
func (PageAdded) domainEvent()      {}
func (PageRemoved) domainEvent()    {}
```

### Test checklist

- [ ] Push all event types, assert len and concrete types via type switch
- [ ] ClearEvents resets to empty slice
