# Phase 5 — Serializer + public API

## Step 14a — Serializer primitives (`infra/serializer.go`)

Shared by both save modes.

```go
package infra

// writeObject: serializes one PDF object to w, returns bytes written
// format: "N G obj\n<value>\nendobj\n"
func writeObject(w io.Writer, id domain.ObjectID, obj domain.PdfObject) (int64, error)

// writeXRefSection: writes a classic xref subsection for the given entries
func writeXRefSection(w io.Writer, entries []domain.XRefEntry) error

// writeTrailer: writes trailer dict + startxref + %%EOF
// prevXRefOffset = -1 for full rewrite (no /Prev)
func writeTrailer(w io.Writer, size int, rootID domain.ObjectID, xrefOffset int64, prevXRefOffset int64) error
```

### Test checklist

- [ ] writeObject(dict) → parse back with parseObject(), assert keys match
- [ ] writeObject(stream) → roundtrip Data bytes
- [ ] writeXRefSection: bytes match classic xref format exactly
- [ ] writeTrailer with prevXRefOffset=-1: no /Prev key in output

---

## Step 14b — SaveIncremental (`infra/serializer.go`)

```go
type Serializer struct{}

func (s *Serializer) SaveIncremental(path string, doc *domain.Document) error
// 1. os.OpenFile(path, O_APPEND|O_WRONLY)
// 2. currentOffset = file size (seek end)
// 3. for each event in doc.PendingEvents():
//      ObjectModified → writeObject → record new XRefEntry{offset}
//      ObjectRemoved  → record XRefEntry{InUse: false}
// 4. writeXRefSection(newEntries)
// 5. writeTrailer(size, rootID, xrefOffset, prevXRefOffset)
// 6. doc.ClearEvents()
```

### Test checklist

- [ ] Modify metadata of fixture PDF, SaveIncremental, re-open with OpenReader
- [ ] Assert new /Author value present in Info object
- [ ] Assert original page content still resolves correctly
- [ ] File size is larger than original (appended)

---

## Step 14c — Save full rewrite (`infra/serializer.go`)

```go
func (s *Serializer) Save(path string, doc *domain.Document) error
// 1. Create new file at path
// 2. Write "%PDF-1.7\n%âãÏÓ\n" header
// 3. Iterate doc.XRef().All() in id order
//    - skip free entries
//    - writeObject for each, record offset
// 4. writeXRefSection(all entries with new offsets)
// 5. writeTrailer(size, rootID, xrefOffset, prevXRefOffset=-1)
// 6. doc.ClearEvents()
```

### Test checklist

- [ ] pdf.New() → AddPage → Save → open file, assert valid PDF header
- [ ] Page count correct after round-trip
- [ ] File passes `pdfinfo` without errors

---

## Step 15 — Application layer (`application/editor.go`)

```go
package application

type Editor struct {
    doc        *domain.Document
    reader     *infra.Reader       // nil for New()
    serializer *infra.Serializer
    sourcePath string
}

// Open existing PDF
func Open(path string) (*Editor, error)
// 1. infra.OpenReader(path)
// 2. reader.LoadXRef() → xref, rootID, infoID
// 3. domain.NewDocument(xref, reader)
// 4. doc.RootID = rootID, doc.InfoID = infoID

// Create new empty PDF
func New() *Editor
// domain.NewDocument(NewXRefTable(), nil)
// allocate minimal objects: catalog, page tree

func (e *Editor) Catalog() (*domain.nodes.Catalog, error)
func (e *Editor) SetMetadata(key, value string) error
func (e *Editor) AddPage() (*domain.nodes.Page, error)

func (e *Editor) Save(outPath string) error              // full rewrite
func (e *Editor) SaveIncremental(outPath string) error   // append-only
func (e *Editor) Close() error
```

### Test checklist

- [ ] Open → SetMetadata → SaveIncremental → re-Open → assert metadata
- [ ] New → AddPage → AddComponent → Save → pdfinfo passes
- [ ] Close releases file handle

---

## Step 16 — Public API (`pdf.go`) + e2e tests

```go
package pdf

import "github.com/felipemalacarne/papyrus/application"

type Editor = application.Editor

func Open(path string) (*Editor, error) { return application.Open(path) }
func New() *Editor                      { return application.New() }
```

### End-to-end usage

```go
// Modify existing
ed, _ := pdf.Open("invoice.pdf")
defer ed.Close()
ed.SetMetadata("Author", "Felipe")
page, _ := ed.Catalog().GetPages().GetPage(0)
page.AddComponent(
    components.NewBox("header").
        WithBounds(40, 750, 515, 60).
        WithFill(color.RGB{0.1, 0.4, 0.8}).
        AddChild(
            components.NewText("title").
                WithPosition(50, 768).
                WithFont("/F1", 14).
                WithContent("Updated"),
        ),
)
ed.SaveIncremental("invoice-updated.pdf")

// Create from scratch
ed2 := pdf.New()
page2, _ := ed2.AddPage()
page2.AddComponent(
    components.NewText("hello").
        WithPosition(100, 700).
        WithFont("/F1", 12).
        WithContent("Hello, PDF"),
)
ed2.Save("new.pdf")
```

### E2E test checklist

- [ ] `TestModifyExisting`: open fixture → modify → save incremental → re-open → assert
- [ ] `TestCreateFromScratch`: New → AddPage → AddComponent → Save → pdfinfo
- [ ] `TestRoundtrip`: Open → Save (full rewrite) → Open → assert same page count and metadata
