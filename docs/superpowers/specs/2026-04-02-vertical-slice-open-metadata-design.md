# Design: Vertical Slice — Open PDF + Read Metadata

**Date:** 2026-04-02
**Scope:** First production-quality vertical slice of the papyrus library.
**Goal:** `pdf.Open(path)` → `editor.Metadata()` + `editor.PageCount()` working end-to-end against a real PDF file.

---

## Approach

Build inside-out from the public API stub. Define the API surface first, then fill in layers bottom-up until the integration test passes. Every piece written in this slice is permanent — no throwaway code.

---

## Public API (`pdf.go`)

```go
func Open(path string) (*application.Editor, error)
```

Thin wrapper delegating to `application.Open`.

---

## Application Layer (`application/editor.go`)

```go
type Editor struct {
    doc    *domain.Document
    reader *infra.Reader
}

func Open(path string) (*Editor, error)
func (e *Editor) Metadata() map[string]string
func (e *Editor) PageCount() int
func (e *Editor) Close() error
```

`Open` flow:
1. Call `infra.NewReader(path)` → returns `*Reader`, `*domain.XRefTable`, `domain.PdfDict` (trailer)
2. Call `domain.NewDocument(xref, trailer, reader)` → returns `*Document`
3. Return `&Editor{doc, reader}`

`Metadata()` and `PageCount()` delegate directly to `doc.Metadata()` and `doc.PageCount()`.
`Close()` calls `reader.Close()`.

---

## Domain Layer

### `domain/object.go`

All 10 PDF value types implementing the `PdfObject` marker interface:
`PdfDict`, `PdfArray`, `PdfName`, `PdfString`, `PdfInt`, `PdfReal`, `PdfBool`, `PdfNull`, `PdfRef`, `PdfStream`.

`ObjectID = PdfRef` type alias.

### `domain/xref.go`

```go
type XRefEntry struct {
    ObjID  ObjectID
    Offset int64
    InUse  bool
}

type XRefTable struct { ... }

func NewXRefTable() *XRefTable
func (x *XRefTable) Lookup(id uint32) (XRefEntry, bool)
func (x *XRefTable) Insert(entry XRefEntry)
func (x *XRefTable) MarkFree(id ObjectID)
func (x *XRefTable) NextFreeID() uint32
func (x *XRefTable) All() []XRefEntry
```

### `domain/events.go`

Four event types: `ObjectModified`, `ObjectRemoved`, `PageAdded`, `PageRemoved` — all implementing `DomainEvent`. Not consumed by this slice but completing Phase 1.

### `domain/document.go`

```go
type ObjectResolver interface {
    ReadObjectAt(id ObjectID) (PdfObject, error)
}

type Document struct {
    xref     *XRefTable
    resolver ObjectResolver
    cache    map[uint32]PdfObject
    info     ObjectID   // from trailer /Info
    root     ObjectID   // from trailer /Root
    events   []DomainEvent
}

func NewDocument(xref *XRefTable, trailer PdfDict, resolver ObjectResolver) (*Document, error)
func (d *Document) Resolve(id ObjectID) (PdfObject, error)   // cache-through resolver
func (d *Document) Metadata() map[string]string              // lazy: resolves /Info dict
func (d *Document) PageCount() int                           // lazy: resolves /Root → /Pages → /Count
func (d *Document) PendingEvents() []DomainEvent
func (d *Document) ClearEvents()
```

`Metadata()` resolves `d.info`, casts to `PdfDict`, extracts string values for standard Info keys (`Title`, `Author`, `Subject`, `Keywords`, `Creator`, `Producer`, `CreationDate`, `ModDate`). Returns empty map if `/Info` is absent.

`PageCount()` resolves `d.root` (Catalog dict), follows `/Pages` ref to the PageTree dict, reads `/Count` as `PdfInt`. Returns 0 on any error.

`NodeRef[T]` and typed node structs (`Catalog`, `PageTree`, etc.) are **deferred** to the next phase — `Document` navigates raw dicts for this slice.

---

## Infrastructure Layer (`infra/reader.go`)

```go
type Reader struct {
    f    *os.File
    xref *domain.XRefTable
}

func NewReader(path string) (*Reader, *domain.XRefTable, domain.PdfDict, error)
func (r *Reader) ReadObjectAt(id domain.ObjectID) (domain.PdfObject, error)
func (r *Reader) Close() error
```

### Internal pipeline

**Tokenizer** — operates on `io.ReadSeeker`, emits tokens:
- Scalars: `Name`, `String` (literal + hex), `Int`, `Real`, `Bool`, `Null`
- Delimiters: `ArrayStart`, `ArrayEnd`, `DictStart`, `DictEnd`
- Keywords: `obj`, `endobj`, `stream`, `endstream`, `xref`, `trailer`, `startxref`

**Object parser** — consumes tokens, builds `PdfObject` values. Handles nested dicts/arrays, indirect object headers (`n g obj … endobj`), stream bodies (reads `/Length` bytes raw; decoding is out of scope for this slice).

**XRef loader** — startup sequence:
1. Seek to last 1024 bytes, find `startxref`, read offset
2. Seek to that offset, parse classic xref table (subsections of `n g offset` lines)
3. Parse `trailer` dict
4. Populate `XRefTable`
5. Return xref + trailer

`infra/decoder.go` (FlateDecode) is **not in scope** for this slice — the Info dict and PageTree root are always plain uncompressed dicts.

---

## Testing

### Unit tests (no I/O)

| Test file | What it covers |
|-----------|---------------|
| `domain/object_test.go` | Type switch on each concrete type; `PdfDict` key lookup; `PdfRef` equality |
| `domain/xref_test.go` | Insert 10 entries + Lookup; MarkFree; NextFreeID; All() ordering |
| `domain/events_test.go` | Push all 4 event types; ClearEvents |
| `domain/document_test.go` | Mock `ObjectResolver`; assert `Metadata()` extracts correct keys; assert `PageCount()` follows the right refs |

### Integration tests (real PDFs)

| Test file | What it covers |
|-----------|---------------|
| `infra/reader_test.go` | Parse `testdata/simple.pdf`: xref entry count, trailer has `/Root` and `/Info` |
| `infra/reader_test.go` | `ReadObjectAt` for a known object ID returns expected dict |
| `pdf_test.go` | `pdf.Open("testdata/simple.pdf")` → `PageCount() == 1`, `Metadata()` returns non-empty map |

### Test fixture

`testdata/simple.pdf` — 1 page, no images, classic xref table, no encryption, has an Info dict with at least `Title`. Generated with Ghostscript or LibreOffice and committed.

---

## Out of Scope for This Slice

- `NodeRef[T]` and typed node types (`Catalog`, `PageTree`, `Page`, etc.)
- `infra/decoder.go` (FlateDecode, ASCIIHexDecode)
- `infra/serializer.go` (Save, SaveIncremental)
- `components/` package
- Any mutation operations
- XRef stream parsing (PDF 1.5+ compressed xref)
