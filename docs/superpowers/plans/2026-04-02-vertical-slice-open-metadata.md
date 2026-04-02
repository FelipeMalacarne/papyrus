# Vertical Slice: Open PDF + Read Metadata Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `pdf.Open(path)` → `editor.Metadata()` + `editor.PageCount()` working end-to-end against a real PDF file.

**Architecture:** Build inside-out from the public API stub. Define stubs first, then fill layers bottom-up: domain types → tokenizer → object parser → xref loader → application editor → public API. The `domain.ObjectResolver` interface takes an `offset int64` (not an ObjectID); `Document.Resolve` handles the xref lookup and calls the resolver with the raw file offset.

**Tech Stack:** Go 1.24.3, stdlib only (`bufio`, `bytes`, `fmt`, `io`, `os`, `strconv`, `strings`).

---

## File Map

| File | Package | Responsibility |
|------|---------|----------------|
| `domain/object.go` | `domain` | All 10 PdfObject types + marker interface |
| `domain/xref.go` | `domain` | XRefTable: in-memory offset index |
| `domain/events.go` | `domain` | 4 DomainEvent types |
| `domain/document.go` | `domain` | Aggregate root: cache-through resolver, Metadata, PageCount |
| `infra/reader.go` | `infra` | Tokenizer, object parser, xref loader, Reader |
| `application/editor.go` | `application` | Open, Metadata, PageCount, Close |
| `pdf.go` | `papyrus` | Public `Open` wrapper |
| `scripts/create_fixtures.py` | — | Generates `testdata/simple.pdf` |

---

## Task 1: PdfObject types

**Files:**
- Create: `domain/object.go`
- Create: `domain/object_test.go`

- [ ] **Step 1: Write the failing test**

```go
// domain/object_test.go
package domain

import "testing"

func TestPdfObjectTypeSwitch(t *testing.T) {
	objects := []PdfObject{
		PdfDict{"k": PdfString("v")},
		PdfArray{PdfInt(1)},
		PdfName("Name"),
		PdfString("str"),
		PdfInt(42),
		PdfReal(3.14),
		PdfBool(true),
		PdfNull{},
		PdfRef{ID: 1, Gen: 0},
		PdfStream{Dict: PdfDict{}, Data: []byte("d")},
	}
	want := []string{"dict", "array", "name", "string", "int", "real", "bool", "null", "ref", "stream"}
	for i, obj := range objects {
		var got string
		switch obj.(type) {
		case PdfDict:   got = "dict"
		case PdfArray:  got = "array"
		case PdfName:   got = "name"
		case PdfString: got = "string"
		case PdfInt:    got = "int"
		case PdfReal:   got = "real"
		case PdfBool:   got = "bool"
		case PdfNull:   got = "null"
		case PdfRef:    got = "ref"
		case PdfStream: got = "stream"
		}
		if got != want[i] {
			t.Errorf("object %d: want %s got %s", i, want[i], got)
		}
	}
}

func TestPdfDictLookup(t *testing.T) {
	d := PdfDict{"Type": PdfName("Page"), "Count": PdfInt(3)}
	if v := d["Type"]; v != PdfName("Page") {
		t.Errorf("Type: got %v", v)
	}
	if v := d["Count"]; v != PdfInt(3) {
		t.Errorf("Count: got %v", v)
	}
	if _, ok := d["Missing"]; ok {
		t.Error("expected missing key to be absent")
	}
}

func TestPdfRefEquality(t *testing.T) {
	a := PdfRef{ID: 5, Gen: 0}
	b := PdfRef{ID: 5, Gen: 0}
	c := PdfRef{ID: 5, Gen: 1}
	if a != b {
		t.Error("same ID+Gen should be equal")
	}
	if a == c {
		t.Error("different Gen should not be equal")
	}
}
```

- [ ] **Step 2: Run test — verify it fails**

```
go test ./domain/... -run TestPdfObject
```
Expected: compile error (types not defined yet).

- [ ] **Step 3: Implement**

```go
// domain/object.go
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

- [ ] **Step 4: Run test — verify it passes**

```
go test ./domain/... -run TestPdf
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add domain/object.go domain/object_test.go
git commit -m "feat: add PdfObject type system"
```

---

## Task 2: XRefTable

**Files:**
- Create: `domain/xref.go`
- Create: `domain/xref_test.go`

- [ ] **Step 1: Write the failing test**

```go
// domain/xref_test.go
package domain

import "testing"

func TestXRefTableInsertLookup(t *testing.T) {
	x := NewXRefTable()
	for i := uint32(1); i <= 10; i++ {
		x.Insert(XRefEntry{
			ObjID:  ObjectID{ID: i, Gen: 0},
			Offset: int64(i * 100),
			InUse:  true,
		})
	}
	for i := uint32(1); i <= 10; i++ {
		e, ok := x.Lookup(i)
		if !ok {
			t.Fatalf("Lookup(%d): not found", i)
		}
		if e.Offset != int64(i*100) {
			t.Errorf("Lookup(%d): offset want %d got %d", i, i*100, e.Offset)
		}
	}
}

func TestXRefTableMarkFree(t *testing.T) {
	x := NewXRefTable()
	x.Insert(XRefEntry{ObjID: ObjectID{ID: 1, Gen: 0}, Offset: 10, InUse: true})
	x.MarkFree(ObjectID{ID: 1, Gen: 0})
	e, ok := x.Lookup(1)
	if !ok {
		t.Fatal("entry should still exist after MarkFree")
	}
	if e.InUse {
		t.Error("InUse should be false after MarkFree")
	}
}

func TestXRefTableNextFreeID(t *testing.T) {
	x := NewXRefTable()
	x.Insert(XRefEntry{ObjID: ObjectID{ID: 1, Gen: 0}, Offset: 10, InUse: true})
	x.Insert(XRefEntry{ObjID: ObjectID{ID: 2, Gen: 0}, Offset: 20, InUse: true})
	// no free slots → next is max+1 = 3
	if id := x.NextFreeID(); id != 3 {
		t.Errorf("NextFreeID: want 3 got %d", id)
	}
	x.MarkFree(ObjectID{ID: 1})
	// free slot exists → should return 1
	if id := x.NextFreeID(); id != 1 {
		t.Errorf("NextFreeID (free slot): want 1 got %d", id)
	}
}

func TestXRefTableAll(t *testing.T) {
	x := NewXRefTable()
	x.Insert(XRefEntry{ObjID: ObjectID{ID: 3, Gen: 0}, Offset: 30, InUse: true})
	x.Insert(XRefEntry{ObjID: ObjectID{ID: 1, Gen: 0}, Offset: 10, InUse: true})
	x.Insert(XRefEntry{ObjID: ObjectID{ID: 2, Gen: 0}, Offset: 20, InUse: false}) // free
	all := x.All()
	if len(all) != 2 {
		t.Fatalf("All: want 2 in-use entries, got %d", len(all))
	}
	if all[0].ObjID.ID != 1 || all[1].ObjID.ID != 3 {
		t.Errorf("All: want sorted by id, got %d %d", all[0].ObjID.ID, all[1].ObjID.ID)
	}
}
```

- [ ] **Step 2: Run test — verify it fails**

```
go test ./domain/... -run TestXRef
```
Expected: compile error.

- [ ] **Step 3: Implement**

```go
// domain/xref.go
package domain

import "sort"

type XRefEntry struct {
	ObjID  ObjectID
	Offset int64
	InUse  bool
}

type XRefTable struct {
	entries map[uint32]XRefEntry
	maxID   uint32
}

func NewXRefTable() *XRefTable {
	return &XRefTable{entries: make(map[uint32]XRefEntry)}
}

func (x *XRefTable) Lookup(id uint32) (XRefEntry, bool) {
	e, ok := x.entries[id]
	return e, ok
}

func (x *XRefTable) Insert(entry XRefEntry) {
	x.entries[entry.ObjID.ID] = entry
	if entry.ObjID.ID > x.maxID {
		x.maxID = entry.ObjID.ID
	}
}

func (x *XRefTable) MarkFree(id ObjectID) {
	if e, ok := x.entries[id.ID]; ok {
		e.InUse = false
		x.entries[id.ID] = e
	}
}

func (x *XRefTable) NextFreeID() uint32 {
	for id, e := range x.entries {
		if !e.InUse {
			return id
		}
	}
	return x.maxID + 1
}

func (x *XRefTable) All() []XRefEntry {
	result := make([]XRefEntry, 0, len(x.entries))
	for _, e := range x.entries {
		if e.InUse {
			result = append(result, e)
		}
	}
	sort.Slice(result, func(i, j int) bool {
		return result[i].ObjID.ID < result[j].ObjID.ID
	})
	return result
}
```

- [ ] **Step 4: Run test — verify it passes**

```
go test ./domain/... -run TestXRef
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add domain/xref.go domain/xref_test.go
git commit -m "feat: add XRefTable"
```

---

## Task 3: DomainEvent types

**Files:**
- Create: `domain/events.go`
- Create: `domain/events_test.go`

- [ ] **Step 1: Write the failing test**

```go
// domain/events_test.go
package domain

import "testing"

func TestDomainEvents(t *testing.T) {
	events := []DomainEvent{
		ObjectModified{ObjID: ObjectID{ID: 1}, NewValue: PdfNull{}},
		ObjectRemoved{ObjID: ObjectID{ID: 2}},
		PageAdded{PageNum: 1},
		PageRemoved{PageNum: 1},
	}
	want := []string{"modified", "removed", "pageAdded", "pageRemoved"}
	for i, e := range events {
		var got string
		switch e.(type) {
		case ObjectModified: got = "modified"
		case ObjectRemoved:  got = "removed"
		case PageAdded:      got = "pageAdded"
		case PageRemoved:    got = "pageRemoved"
		}
		if got != want[i] {
			t.Errorf("event %d: want %s got %s", i, want[i], got)
		}
	}
}
```

- [ ] **Step 2: Run test — verify it fails**

```
go test ./domain/... -run TestDomainEvents
```
Expected: compile error.

- [ ] **Step 3: Implement**

```go
// domain/events.go
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

- [ ] **Step 4: Run test — verify it passes**

```
go test ./domain/... -run TestDomainEvents
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add domain/events.go domain/events_test.go
git commit -m "feat: add DomainEvent types"
```

---

## Task 4: Document aggregate

**Files:**
- Create: `domain/document.go`
- Create: `domain/document_test.go`

- [ ] **Step 1: Write the failing test**

```go
// domain/document_test.go
package domain

import (
	"fmt"
	"testing"
)

type mockResolver struct {
	objects map[uint32]PdfObject
	calls   int
}

func (m *mockResolver) ReadObjectAt(offset int64) (PdfObject, error) {
	m.calls++
	// In tests, we use the offset as the object ID for simplicity
	obj, ok := m.objects[uint32(offset)]
	if !ok {
		return nil, fmt.Errorf("object at offset %d not found", offset)
	}
	return obj, nil
}

func newTestDoc(t *testing.T) (*Document, *mockResolver) {
	t.Helper()
	xref := NewXRefTable()
	// Object 1 = Catalog (offset = 1)
	xref.Insert(XRefEntry{ObjID: ObjectID{ID: 1}, Offset: 1, InUse: true})
	// Object 2 = PageTree (offset = 2)
	xref.Insert(XRefEntry{ObjID: ObjectID{ID: 2}, Offset: 2, InUse: true})
	// Object 3 = Info (offset = 3)
	xref.Insert(XRefEntry{ObjID: ObjectID{ID: 3}, Offset: 3, InUse: true})

	mock := &mockResolver{
		objects: map[uint32]PdfObject{
			1: PdfDict{"Type": PdfName("Catalog"), "Pages": PdfRef{ID: 2}},
			2: PdfDict{"Type": PdfName("Pages"), "Count": PdfInt(2)},
			3: PdfDict{"Title": PdfString("Test Doc"), "Author": PdfString("Jane")},
		},
	}
	doc := NewDocument(xref, mock)
	doc.RootID = ObjectID{ID: 1}
	doc.InfoID = ObjectID{ID: 3}
	return doc, mock
}

func TestDocumentResolveCachesResult(t *testing.T) {
	doc, mock := newTestDoc(t)
	_, err := doc.Resolve(ObjectID{ID: 1})
	if err != nil {
		t.Fatalf("Resolve: %v", err)
	}
	_, err = doc.Resolve(ObjectID{ID: 1})
	if err != nil {
		t.Fatalf("Resolve second call: %v", err)
	}
	if mock.calls != 1 {
		t.Errorf("ReadObjectAt called %d times, want 1 (cache miss only)", mock.calls)
	}
}

func TestDocumentMetadata(t *testing.T) {
	doc, _ := newTestDoc(t)
	meta := doc.Metadata()
	if meta["Title"] != "Test Doc" {
		t.Errorf("Title: want %q got %q", "Test Doc", meta["Title"])
	}
	if meta["Author"] != "Jane" {
		t.Errorf("Author: want %q got %q", "Jane", meta["Author"])
	}
}

func TestDocumentMetadataAbsent(t *testing.T) {
	xref := NewXRefTable()
	xref.Insert(XRefEntry{ObjID: ObjectID{ID: 1}, Offset: 1, InUse: true})
	mock := &mockResolver{
		objects: map[uint32]PdfObject{
			1: PdfDict{"Type": PdfName("Catalog"), "Pages": PdfRef{ID: 2}},
		},
	}
	doc := NewDocument(xref, mock)
	doc.RootID = ObjectID{ID: 1}
	// InfoID is zero value — no info dict
	meta := doc.Metadata()
	if len(meta) != 0 {
		t.Errorf("expected empty metadata when InfoID is zero, got %v", meta)
	}
}

func TestDocumentPageCount(t *testing.T) {
	doc, _ := newTestDoc(t)
	if n := doc.PageCount(); n != 2 {
		t.Errorf("PageCount: want 2 got %d", n)
	}
}

func TestDocumentPendingEventsAndClear(t *testing.T) {
	doc, _ := newTestDoc(t)
	if len(doc.PendingEvents()) != 0 {
		t.Error("expected no events initially")
	}
	doc.events = append(doc.events, PageAdded{PageNum: 1})
	if len(doc.PendingEvents()) != 1 {
		t.Error("expected 1 event after append")
	}
	doc.ClearEvents()
	if len(doc.PendingEvents()) != 0 {
		t.Error("expected 0 events after ClearEvents")
	}
}
```

- [ ] **Step 2: Run test — verify it fails**

```
go test ./domain/... -run TestDocument
```
Expected: compile error.

- [ ] **Step 3: Implement**

```go
// domain/document.go
package domain

import "fmt"

// ObjectResolver is implemented by infra.Reader and injected into Document.
// It takes the byte offset of an indirect object in the file.
type ObjectResolver interface {
	ReadObjectAt(offset int64) (PdfObject, error)
}

type Document struct {
	xref     *XRefTable
	cache    map[uint32]PdfObject
	events   []DomainEvent
	resolver ObjectResolver

	// Set by the application layer after parsing the trailer.
	RootID ObjectID
	InfoID ObjectID
}

func NewDocument(xref *XRefTable, resolver ObjectResolver) *Document {
	return &Document{
		xref:     xref,
		cache:    make(map[uint32]PdfObject),
		events:   []DomainEvent{},
		resolver: resolver,
	}
}

// Resolve returns the PdfObject for id, using the cache to avoid redundant I/O.
func (d *Document) Resolve(id ObjectID) (PdfObject, error) {
	if obj, ok := d.cache[id.ID]; ok {
		return obj, nil
	}
	entry, ok := d.xref.Lookup(id.ID)
	if !ok {
		return nil, fmt.Errorf("object %d not in xref table", id.ID)
	}
	obj, err := d.resolver.ReadObjectAt(entry.Offset)
	if err != nil {
		return nil, fmt.Errorf("reading object %d: %w", id.ID, err)
	}
	d.cache[id.ID] = obj
	return obj, nil
}

// Metadata returns string values from the Info dictionary.
// Returns an empty map if no Info dict is present.
func (d *Document) Metadata() map[string]string {
	result := make(map[string]string)
	if d.InfoID.ID == 0 {
		return result
	}
	obj, err := d.Resolve(d.InfoID)
	if err != nil {
		return result
	}
	dict, ok := obj.(PdfDict)
	if !ok {
		return result
	}
	for _, k := range []string{"Title", "Author", "Subject", "Keywords", "Creator", "Producer", "CreationDate", "ModDate"} {
		if v, ok := dict[k]; ok {
			if s, ok := v.(PdfString); ok {
				result[k] = string(s)
			}
		}
	}
	return result
}

// PageCount follows /Root → Catalog → /Pages → PageTree → /Count.
// Returns 0 on any error.
func (d *Document) PageCount() int {
	catalogObj, err := d.Resolve(d.RootID)
	if err != nil {
		return 0
	}
	catalog, ok := catalogObj.(PdfDict)
	if !ok {
		return 0
	}
	pagesRefObj, ok := catalog["Pages"]
	if !ok {
		return 0
	}
	pagesRef, ok := pagesRefObj.(PdfRef)
	if !ok {
		return 0
	}
	pagesObj, err := d.Resolve(pagesRef)
	if err != nil {
		return 0
	}
	pagesDict, ok := pagesObj.(PdfDict)
	if !ok {
		return 0
	}
	countObj, ok := pagesDict["Count"]
	if !ok {
		return 0
	}
	count, ok := countObj.(PdfInt)
	if !ok {
		return 0
	}
	return int(count)
}

func (d *Document) PendingEvents() []DomainEvent { return d.events }
func (d *Document) ClearEvents()                 { d.events = d.events[:0] }
```

- [ ] **Step 4: Run test — verify it passes**

```
go test ./domain/...
```
Expected: PASS (all domain tests)

- [ ] **Step 5: Commit**

```bash
git add domain/document.go domain/document_test.go
git commit -m "feat: add Document aggregate with Resolve, Metadata, PageCount"
```

---

## Task 5: Tokenizer

**Files:**
- Create: `infra/reader.go` (tokenizer portion)
- Create: `infra/reader_test.go` (tokenizer tests)

- [ ] **Step 1: Write the failing test**

```go
// infra/reader_test.go
package infra

import (
	"strings"
	"testing"
)

func tok(t *testing.T, input string) []token {
	t.Helper()
	tz := newTokenizer(strings.NewReader(input))
	var tokens []token
	for {
		tok, err := tz.next()
		if err != nil {
			t.Fatalf("tokenizer error: %v", err)
		}
		if tok.kind == tokEOF {
			break
		}
		tokens = append(tokens, tok)
	}
	return tokens
}

func TestTokenizerIntegers(t *testing.T) {
	tokens := tok(t, "42 -7 0")
	if len(tokens) != 3 {
		t.Fatalf("want 3 tokens got %d", len(tokens))
	}
	for i, want := range []string{"42", "-7", "0"} {
		if tokens[i].kind != tokInt {
			t.Errorf("token %d: want tokInt", i)
		}
		if string(tokens[i].raw) != want {
			t.Errorf("token %d raw: want %q got %q", i, want, tokens[i].raw)
		}
	}
}

func TestTokenizerReal(t *testing.T) {
	tokens := tok(t, "3.14 -0.5")
	for i, tc := range []struct{ raw string }{{"3.14"}, {"-0.5"}} {
		if tokens[i].kind != tokReal {
			t.Errorf("token %d: want tokReal", i)
		}
		if string(tokens[i].raw) != tc.raw {
			t.Errorf("token %d raw: want %q got %q", i, tc.raw, tokens[i].raw)
		}
	}
}

func TestTokenizerBoolNull(t *testing.T) {
	tokens := tok(t, "true false null")
	if tokens[0].kind != tokBool || string(tokens[0].raw) != "true" {
		t.Errorf("want tokBool true, got kind=%d raw=%q", tokens[0].kind, tokens[0].raw)
	}
	if tokens[1].kind != tokBool || string(tokens[1].raw) != "false" {
		t.Errorf("want tokBool false, got kind=%d raw=%q", tokens[1].kind, tokens[1].raw)
	}
	if tokens[2].kind != tokNull {
		t.Errorf("want tokNull, got kind=%d", tokens[2].kind)
	}
}

func TestTokenizerName(t *testing.T) {
	tokens := tok(t, "/Type /MediaBox")
	for i, want := range []string{"Type", "MediaBox"} {
		if tokens[i].kind != tokName {
			t.Errorf("token %d: want tokName", i)
		}
		if string(tokens[i].raw) != want {
			t.Errorf("token %d raw: want %q got %q", i, want, tokens[i].raw)
		}
	}
}

func TestTokenizerLiteralString(t *testing.T) {
	tokens := tok(t, "(hello world)")
	if len(tokens) != 1 || tokens[0].kind != tokString {
		t.Fatalf("want 1 tokString, got %d tokens", len(tokens))
	}
	if string(tokens[0].raw) != "hello world" {
		t.Errorf("raw: want %q got %q", "hello world", tokens[0].raw)
	}
}

func TestTokenizerLiteralStringEscape(t *testing.T) {
	tokens := tok(t, `(\n\t\\)`)
	if string(tokens[0].raw) != "\n\t\\" {
		t.Errorf("escape: got %q", tokens[0].raw)
	}
}

func TestTokenizerHexString(t *testing.T) {
	tokens := tok(t, "<68656c6c6f>")
	if len(tokens) != 1 || tokens[0].kind != tokString {
		t.Fatalf("want 1 tokString")
	}
	if string(tokens[0].raw) != "hello" {
		t.Errorf("hex string: want %q got %q", "hello", tokens[0].raw)
	}
}

func TestTokenizerDictDelimiters(t *testing.T) {
	tokens := tok(t, "<< >>")
	if tokens[0].kind != tokDictOpen {
		t.Errorf("want tokDictOpen")
	}
	if tokens[1].kind != tokDictClose {
		t.Errorf("want tokDictClose")
	}
}

func TestTokenizerArrayDelimiters(t *testing.T) {
	tokens := tok(t, "[ ]")
	if tokens[0].kind != tokArrayOpen {
		t.Errorf("want tokArrayOpen")
	}
	if tokens[1].kind != tokArrayClose {
		t.Errorf("want tokArrayClose")
	}
}

func TestTokenizerKeywords(t *testing.T) {
	for _, kw := range []string{"obj", "endobj", "stream", "endstream", "xref", "trailer", "startxref", "R", "n", "f"} {
		tokens := tok(t, kw)
		if len(tokens) != 1 || tokens[0].kind != tokKeyword {
			t.Errorf("keyword %q: want 1 tokKeyword, got kind=%d", kw, tokens[0].kind)
		}
		if string(tokens[0].raw) != kw {
			t.Errorf("keyword %q: raw mismatch %q", kw, tokens[0].raw)
		}
	}
}

func TestTokenizerSkipsComments(t *testing.T) {
	tokens := tok(t, "% this is a comment\n42")
	if len(tokens) != 1 || tokens[0].kind != tokInt || string(tokens[0].raw) != "42" {
		t.Errorf("want single tokInt 42 after comment, got %v", tokens)
	}
}

func TestTokenizerUnget(t *testing.T) {
	tz := newTokenizer(strings.NewReader("1 2 3"))
	t1, _ := tz.next()
	t2, _ := tz.next()
	tz.unget(t2)
	tz.unget(t1)
	r1, _ := tz.next()
	r2, _ := tz.next()
	if string(r1.raw) != "1" || string(r2.raw) != "2" {
		t.Errorf("unget order wrong: got %q %q", r1.raw, r2.raw)
	}
}
```

- [ ] **Step 2: Run test — verify it fails**

```
go test ./infra/... -run TestTokenizer
```
Expected: compile error.

- [ ] **Step 3: Implement the tokenizer**

```go
// infra/reader.go
package infra

import (
	"bufio"
	"bytes"
	"fmt"
	"io"
	"os"
	"strconv"

	"github.com/felipemalacarne/papyrus/domain"
)

// ─── Tokenizer ────────────────────────────────────────────────────────────────

type tokenKind int

const (
	tokInt tokenKind = iota
	tokReal
	tokBool
	tokNull
	tokName
	tokString
	tokArrayOpen
	tokArrayClose
	tokDictOpen
	tokDictClose
	tokKeyword
	tokEOF
)

type token struct {
	kind tokenKind
	raw  []byte
}

type tokenizer struct {
	r       *bufio.Reader
	pending []token // LIFO stack; last element is next to return
}

func newTokenizer(r io.Reader) *tokenizer {
	return &tokenizer{r: bufio.NewReader(r)}
}

func (t *tokenizer) unget(tok token) {
	t.pending = append(t.pending, tok)
}

func isWhitespace(b byte) bool {
	return b == 0x00 || b == 0x09 || b == 0x0A || b == 0x0C || b == 0x0D || b == 0x20
}

func isDelimiter(b byte) bool {
	switch b {
	case '(', ')', '<', '>', '[', ']', '{', '}', '/', '%':
		return true
	}
	return false
}

func (t *tokenizer) skipWhitespaceAndComments() error {
	for {
		b, err := t.r.ReadByte()
		if err != nil {
			return err
		}
		if b == '%' {
			for {
				c, err := t.r.ReadByte()
				if err != nil {
					return err
				}
				if c == '\n' || c == '\r' {
					break
				}
			}
			continue
		}
		if !isWhitespace(b) {
			_ = t.r.UnreadByte()
			return nil
		}
	}
}

func (t *tokenizer) next() (token, error) {
	if len(t.pending) > 0 {
		tok := t.pending[len(t.pending)-1]
		t.pending = t.pending[:len(t.pending)-1]
		return tok, nil
	}

	if err := t.skipWhitespaceAndComments(); err != nil {
		if err == io.EOF {
			return token{kind: tokEOF}, nil
		}
		return token{}, err
	}

	b, err := t.r.ReadByte()
	if err != nil {
		if err == io.EOF {
			return token{kind: tokEOF}, nil
		}
		return token{}, err
	}

	switch {
	case b == '/':
		return t.readName()
	case b == '(':
		return t.readLiteralString()
	case b == '<':
		next, err := t.r.ReadByte()
		if err != nil {
			return token{}, err
		}
		if next == '<' {
			return token{kind: tokDictOpen, raw: []byte("<<")}, nil
		}
		_ = t.r.UnreadByte()
		return t.readHexString()
	case b == '>':
		next, err := t.r.ReadByte()
		if err != nil {
			return token{}, err
		}
		if next == '>' {
			return token{kind: tokDictClose, raw: []byte(">>")}, nil
		}
		_ = t.r.UnreadByte()
		return token{}, fmt.Errorf("unexpected single '>'")
	case b == '[':
		return token{kind: tokArrayOpen, raw: []byte("[")}, nil
	case b == ']':
		return token{kind: tokArrayClose, raw: []byte("]")}, nil
	case b == '+' || b == '-' || (b >= '0' && b <= '9') || b == '.':
		_ = t.r.UnreadByte()
		return t.readNumber()
	default:
		_ = t.r.UnreadByte()
		return t.readKeyword()
	}
}

func (t *tokenizer) readName() (token, error) {
	var buf bytes.Buffer
	for {
		b, err := t.r.ReadByte()
		if err != nil {
			if err == io.EOF {
				break
			}
			return token{}, err
		}
		if isWhitespace(b) || isDelimiter(b) {
			_ = t.r.UnreadByte()
			break
		}
		buf.WriteByte(b)
	}
	return token{kind: tokName, raw: buf.Bytes()}, nil
}

func (t *tokenizer) readLiteralString() (token, error) {
	var buf bytes.Buffer
	depth := 1
	for depth > 0 {
		b, err := t.r.ReadByte()
		if err != nil {
			return token{}, fmt.Errorf("unterminated literal string: %w", err)
		}
		if b == '\\' {
			next, err := t.r.ReadByte()
			if err != nil {
				return token{}, err
			}
			switch next {
			case 'n':  buf.WriteByte('\n')
			case 'r':  buf.WriteByte('\r')
			case 't':  buf.WriteByte('\t')
			case 'b':  buf.WriteByte('\b')
			case 'f':  buf.WriteByte('\f')
			case '(':  buf.WriteByte('(')
			case ')':  buf.WriteByte(')')
			case '\\': buf.WriteByte('\\')
			default:   buf.WriteByte(next)
			}
			continue
		}
		if b == '(' {
			depth++
			buf.WriteByte(b)
			continue
		}
		if b == ')' {
			depth--
			if depth > 0 {
				buf.WriteByte(b)
			}
			continue
		}
		buf.WriteByte(b)
	}
	return token{kind: tokString, raw: buf.Bytes()}, nil
}

func (t *tokenizer) readHexString() (token, error) {
	var hexBuf bytes.Buffer
	for {
		b, err := t.r.ReadByte()
		if err != nil {
			return token{}, fmt.Errorf("unterminated hex string: %w", err)
		}
		if b == '>' {
			break
		}
		if !isWhitespace(b) {
			hexBuf.WriteByte(b)
		}
	}
	h := hexBuf.Bytes()
	if len(h)%2 != 0 {
		h = append(h, '0')
	}
	out := make([]byte, len(h)/2)
	for i := range out {
		var v byte
		if _, err := fmt.Sscanf(string(h[i*2:i*2+2]), "%02x", &v); err != nil {
			return token{}, fmt.Errorf("invalid hex in string: %w", err)
		}
		out[i] = v
	}
	return token{kind: tokString, raw: out}, nil
}

func (t *tokenizer) readNumber() (token, error) {
	var buf bytes.Buffer
	isReal := false
	for {
		b, err := t.r.ReadByte()
		if err != nil {
			if err == io.EOF {
				break
			}
			return token{}, err
		}
		if b == '.' {
			isReal = true
			buf.WriteByte(b)
			continue
		}
		if (b >= '0' && b <= '9') || b == '+' || b == '-' {
			buf.WriteByte(b)
			continue
		}
		_ = t.r.UnreadByte()
		break
	}
	if isReal {
		return token{kind: tokReal, raw: buf.Bytes()}, nil
	}
	return token{kind: tokInt, raw: buf.Bytes()}, nil
}

func (t *tokenizer) readKeyword() (token, error) {
	var buf bytes.Buffer
	for {
		b, err := t.r.ReadByte()
		if err != nil {
			if err == io.EOF {
				break
			}
			return token{}, err
		}
		if isWhitespace(b) || isDelimiter(b) {
			_ = t.r.UnreadByte()
			break
		}
		buf.WriteByte(b)
	}
	raw := buf.Bytes()
	switch string(raw) {
	case "true":
		return token{kind: tokBool, raw: raw}, nil
	case "false":
		return token{kind: tokBool, raw: raw}, nil
	case "null":
		return token{kind: tokNull, raw: raw}, nil
	}
	return token{kind: tokKeyword, raw: raw}, nil
}

// skipEOL skips the single CR, LF, or CRLF that follows the "stream" keyword.
func (t *tokenizer) skipEOL() error {
	b, err := t.r.ReadByte()
	if err != nil {
		return nil // EOF is acceptable
	}
	if b == '\r' {
		next, err := t.r.ReadByte()
		if err != nil || next != '\n' {
			if err == nil {
				_ = t.r.UnreadByte()
			}
		}
		return nil
	}
	if b != '\n' {
		_ = t.r.UnreadByte()
	}
	return nil
}

// ─── Reader (stub — completed in Tasks 8–9) ──────────────────────────────────

// Reader is declared here so infra compiles; fields added in Task 8.
type Reader struct {
	f *os.File
}

func (r *Reader) Close() error { return r.f.Close() }
```

- [ ] **Step 4: Run test — verify it passes**

```
go test ./infra/... -run TestTokenizer
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add infra/reader.go infra/reader_test.go
git commit -m "feat: add PDF tokenizer"
```

---

## Task 6: Object parser

**Files:**
- Modify: `infra/reader.go` (add parser below the tokenizer section)
- Modify: `infra/reader_test.go` (add parser tests)

- [ ] **Step 1: Add the failing parser tests**

Append to `infra/reader_test.go`:

```go
// ─── Parser tests ─────────────────────────────────────────────────────────────

func parse(t *testing.T, input string) domain.PdfObject {
	t.Helper()
	p := newParser(strings.NewReader(input))
	obj, err := p.parseObject()
	if err != nil {
		t.Fatalf("parseObject(%q): %v", input, err)
	}
	return obj
}

func TestParserInt(t *testing.T) {
	obj := parse(t, "42")
	if obj != domain.PdfInt(42) {
		t.Errorf("want PdfInt(42) got %v", obj)
	}
}

func TestParserReal(t *testing.T) {
	obj := parse(t, "3.14")
	v, ok := obj.(domain.PdfReal)
	if !ok || v < 3.13 || v > 3.15 {
		t.Errorf("want PdfReal(3.14) got %v", obj)
	}
}

func TestParserBoolNull(t *testing.T) {
	if parse(t, "true") != domain.PdfBool(true) {
		t.Error("want PdfBool(true)")
	}
	if parse(t, "false") != domain.PdfBool(false) {
		t.Error("want PdfBool(false)")
	}
	if _, ok := parse(t, "null").(domain.PdfNull); !ok {
		t.Error("want PdfNull")
	}
}

func TestParserName(t *testing.T) {
	if parse(t, "/Page") != domain.PdfName("Page") {
		t.Error("want PdfName(Page)")
	}
}

func TestParserString(t *testing.T) {
	if parse(t, "(hello)") != domain.PdfString("hello") {
		t.Error("want PdfString(hello)")
	}
}

func TestParserIndirectRef(t *testing.T) {
	obj := parse(t, "1 0 R")
	ref, ok := obj.(domain.PdfRef)
	if !ok || ref.ID != 1 || ref.Gen != 0 {
		t.Errorf("want PdfRef{1,0} got %v", obj)
	}
}

func TestParserArray(t *testing.T) {
	obj := parse(t, "[1 2 3]")
	arr, ok := obj.(domain.PdfArray)
	if !ok || len(arr) != 3 {
		t.Fatalf("want PdfArray len 3, got %v", obj)
	}
	if arr[0] != domain.PdfInt(1) || arr[1] != domain.PdfInt(2) || arr[2] != domain.PdfInt(3) {
		t.Errorf("array elements wrong: %v", arr)
	}
}

func TestParserArrayWithRef(t *testing.T) {
	obj := parse(t, "[3 0 R 2]")
	arr, ok := obj.(domain.PdfArray)
	if !ok || len(arr) != 2 {
		t.Fatalf("want PdfArray len 2, got %v", obj)
	}
	if arr[0] != (domain.PdfRef{ID: 3, Gen: 0}) {
		t.Errorf("arr[0]: want PdfRef{3,0} got %v", arr[0])
	}
	if arr[1] != domain.PdfInt(2) {
		t.Errorf("arr[1]: want PdfInt(2) got %v", arr[1])
	}
}

func TestParserDict(t *testing.T) {
	obj := parse(t, "<< /Type /Page /Count 3 >>")
	d, ok := obj.(domain.PdfDict)
	if !ok {
		t.Fatalf("want PdfDict got %T", obj)
	}
	if d["Type"] != domain.PdfName("Page") {
		t.Errorf("Type: got %v", d["Type"])
	}
	if d["Count"] != domain.PdfInt(3) {
		t.Errorf("Count: got %v", d["Count"])
	}
}

func TestParserDictWithRef(t *testing.T) {
	obj := parse(t, "<< /Pages 2 0 R /Info 4 0 R >>")
	d, ok := obj.(domain.PdfDict)
	if !ok {
		t.Fatalf("want PdfDict")
	}
	if d["Pages"] != (domain.PdfRef{ID: 2, Gen: 0}) {
		t.Errorf("Pages: want PdfRef{2,0} got %v", d["Pages"])
	}
}

func TestParserIndirectObject(t *testing.T) {
	p := newParser(strings.NewReader("1 0 obj\n<< /Type /Catalog >>\nendobj"))
	_, obj, err := p.parseIndirectObject()
	if err != nil {
		t.Fatalf("parseIndirectObject: %v", err)
	}
	d, ok := obj.(domain.PdfDict)
	if !ok {
		t.Fatalf("want PdfDict got %T", obj)
	}
	if d["Type"] != domain.PdfName("Catalog") {
		t.Errorf("Type: got %v", d["Type"])
	}
}
```

- [ ] **Step 2: Run tests — verify they fail**

```
go test ./infra/... -run TestParser
```
Expected: compile error (parser not defined).

- [ ] **Step 3: Implement the parser**

Append to `infra/reader.go` (after the tokenizer section):

```go
// ─── Object Parser ────────────────────────────────────────────────────────────

type parser struct {
	tok *tokenizer
}

func newParser(r io.Reader) *parser {
	return &parser{tok: newTokenizer(r)}
}

func (p *parser) parseObject() (domain.PdfObject, error) {
	tok, err := p.tok.next()
	if err != nil {
		return nil, err
	}
	return p.parseValue(tok)
}

func (p *parser) parseValue(tok token) (domain.PdfObject, error) {
	switch tok.kind {
	case tokBool:
		return domain.PdfBool(string(tok.raw) == "true"), nil
	case tokNull:
		return domain.PdfNull{}, nil
	case tokName:
		return domain.PdfName(tok.raw), nil
	case tokString:
		return domain.PdfString(tok.raw), nil
	case tokReal:
		f, err := strconv.ParseFloat(string(tok.raw), 64)
		if err != nil {
			return nil, fmt.Errorf("invalid real %q: %w", tok.raw, err)
		}
		return domain.PdfReal(f), nil
	case tokInt:
		n, err := strconv.ParseInt(string(tok.raw), 10, 64)
		if err != nil {
			return nil, fmt.Errorf("invalid int %q: %w", tok.raw, err)
		}
		// Lookahead: check for indirect reference pattern "n g R"
		tok2, err := p.tok.next()
		if err != nil || tok2.kind != tokInt {
			if err == nil {
				p.tok.unget(tok2)
			}
			return domain.PdfInt(n), nil
		}
		gen, err := strconv.ParseInt(string(tok2.raw), 10, 64)
		if err != nil {
			p.tok.unget(tok2)
			return domain.PdfInt(n), nil
		}
		tok3, err := p.tok.next()
		if err != nil || tok3.kind != tokKeyword || string(tok3.raw) != "R" {
			if err == nil {
				// push back in reverse order (LIFO)
				p.tok.unget(tok3)
			}
			p.tok.unget(tok2)
			return domain.PdfInt(n), nil
		}
		return domain.PdfRef{ID: uint32(n), Gen: uint16(gen)}, nil
	case tokArrayOpen:
		return p.parseArray()
	case tokDictOpen:
		return p.parseDict()
	default:
		return nil, fmt.Errorf("unexpected token kind=%d raw=%q", tok.kind, tok.raw)
	}
}

func (p *parser) parseArray() (domain.PdfArray, error) {
	var arr domain.PdfArray
	for {
		tok, err := p.tok.next()
		if err != nil {
			return nil, err
		}
		if tok.kind == tokArrayClose {
			return arr, nil
		}
		obj, err := p.parseValue(tok)
		if err != nil {
			return nil, err
		}
		arr = append(arr, obj)
	}
}

func (p *parser) parseDict() (domain.PdfDict, error) {
	dict := make(domain.PdfDict)
	for {
		tok, err := p.tok.next()
		if err != nil {
			return nil, err
		}
		if tok.kind == tokDictClose {
			return dict, nil
		}
		if tok.kind != tokName {
			return nil, fmt.Errorf("expected name key in dict, got kind=%d raw=%q", tok.kind, tok.raw)
		}
		key := string(tok.raw)
		val, err := p.parseObject()
		if err != nil {
			return nil, fmt.Errorf("dict value for %q: %w", key, err)
		}
		dict[key] = val
	}
}

// parseIndirectObject parses "n g obj VALUE endobj" and returns the ObjectID + value.
// If the value is a dict followed by "stream", returns a PdfStream.
func (p *parser) parseIndirectObject() (domain.ObjectID, domain.PdfObject, error) {
	t1, err := p.tok.next()
	if err != nil || t1.kind != tokInt {
		return domain.ObjectID{}, nil, fmt.Errorf("expected object number, got kind=%d", t1.kind)
	}
	n, _ := strconv.ParseInt(string(t1.raw), 10, 64)

	t2, err := p.tok.next()
	if err != nil || t2.kind != tokInt {
		return domain.ObjectID{}, nil, fmt.Errorf("expected generation number")
	}
	g, _ := strconv.ParseInt(string(t2.raw), 10, 64)

	t3, err := p.tok.next()
	if err != nil || t3.kind != tokKeyword || string(t3.raw) != "obj" {
		return domain.ObjectID{}, nil, fmt.Errorf("expected 'obj' keyword, got %q", t3.raw)
	}

	id := domain.ObjectID{ID: uint32(n), Gen: uint16(g)}

	obj, err := p.parseObject()
	if err != nil {
		return id, nil, err
	}

	// Check for stream
	if dict, ok := obj.(domain.PdfDict); ok {
		t4, err := p.tok.next()
		if err == nil && t4.kind == tokKeyword && string(t4.raw) == "stream" {
			data, err := p.readStreamData(dict)
			if err != nil {
				return id, nil, err
			}
			return id, domain.PdfStream{Dict: dict, Data: data}, nil
		}
		if err == nil {
			p.tok.unget(t4)
		}
	}

	return id, obj, nil
}

// readStreamData reads /Length bytes from the dict after a "stream" keyword.
// It handles the mandatory single EOL between the keyword and stream data.
func (p *parser) readStreamData(dict domain.PdfDict) ([]byte, error) {
	if err := p.tok.skipEOL(); err != nil {
		return nil, err
	}
	lengthObj, ok := dict["Length"]
	if !ok {
		return nil, fmt.Errorf("stream dict missing /Length")
	}
	length, ok := lengthObj.(domain.PdfInt)
	if !ok {
		return nil, fmt.Errorf("/Length must be a direct integer (indirect references not supported)")
	}
	data := make([]byte, int(length))
	if _, err := io.ReadFull(p.tok.r, data); err != nil {
		return nil, fmt.Errorf("reading stream data: %w", err)
	}
	return data, nil
}
```

- [ ] **Step 4: Run tests — verify they pass**

```
go test ./infra/... -run "TestTokenizer|TestParser"
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add infra/reader.go infra/reader_test.go
git commit -m "feat: add PDF object parser"
```

---

## Task 7: Test fixture

**Files:**
- Create: `scripts/create_fixtures.py`
- Create: `testdata/simple.pdf` (via script)

- [ ] **Step 1: Create the fixture script**

```python
#!/usr/bin/env python3
"""Generate test fixture PDFs for papyrus tests."""

import os


def create_simple_pdf(path: str) -> None:
    """1-page PDF, classic xref table, Info dict with Title and Author."""

    objects = [
        (1, 0, b"<< /Type /Catalog /Pages 2 0 R >>"),
        (2, 0, b"<< /Type /Pages /Kids [3 0 R] /Count 1 >>"),
        (3, 0, b"<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] >>"),
        (4, 0, b"<< /Title (Simple Test) /Author (Test Author) >>"),
    ]

    buf = b"%PDF-1.4\n"
    offsets: dict[int, int] = {}

    for obj_num, gen_num, content in objects:
        offsets[obj_num] = len(buf)
        buf += f"{obj_num} {gen_num} obj\n".encode()
        buf += content + b"\nendobj\n\n"

    xref_offset = len(buf)
    buf += b"xref\n"
    buf += f"0 {len(objects) + 1}\n".encode()
    buf += b"0000000000 65535 f \n"
    for obj_num, gen_num, _ in objects:
        # Each entry is exactly 20 bytes: 10-digit offset + space + 5-digit gen + space + n + space + LF
        buf += f"{offsets[obj_num]:010d} {gen_num:05d} n \n".encode()

    buf += b"trailer\n"
    buf += f"<< /Size {len(objects) + 1} /Root 1 0 R /Info 4 0 R >>\n".encode()
    buf += b"startxref\n"
    buf += f"{xref_offset}\n".encode()
    buf += b"%%EOF\n"

    os.makedirs(os.path.dirname(path) if os.path.dirname(path) else ".", exist_ok=True)
    with open(path, "wb") as f:
        f.write(buf)

    print(f"Created {path} ({len(buf)} bytes)")


if __name__ == "__main__":
    create_simple_pdf("testdata/simple.pdf")
```

- [ ] **Step 2: Run the script**

```
python3 scripts/create_fixtures.py
```
Expected output: `Created testdata/simple.pdf (NNN bytes)`

- [ ] **Step 3: Verify the fixture is readable**

```
cat testdata/simple.pdf
```
Expected: starts with `%PDF-1.4`, ends with `%%EOF`, contains `startxref`.

- [ ] **Step 4: Commit**

```bash
git add scripts/create_fixtures.py testdata/simple.pdf
git commit -m "test: add simple.pdf fixture and generation script"
```

---

## Task 8: XRef loader + NewReader

**Files:**
- Modify: `infra/reader.go` (add xref loader + complete Reader)
- Modify: `infra/reader_test.go` (add integration tests)

- [ ] **Step 1: Update imports in `infra/reader_test.go`**

The integration tests use `domain` types. Update the import block at the top of `infra/reader_test.go`:

```go
import (
	"strings"
	"testing"

	"github.com/felipemalacarne/papyrus/domain"
)
```

- [ ] **Step 2: Add integration tests**

Append to `infra/reader_test.go`:

```go
// ─── Reader integration tests ─────────────────────────────────────────────────

func TestNewReaderLoadsXRef(t *testing.T) {
	r, xref, trailer, err := NewReader("../testdata/simple.pdf")
	if err != nil {
		t.Fatalf("NewReader: %v", err)
	}
	defer r.Close()

	// All 4 in-use objects should be indexed
	all := xref.All()
	if len(all) != 4 {
		t.Errorf("xref.All: want 4 entries, got %d", len(all))
	}

	// Trailer must have /Root and /Info
	if _, ok := trailer["Root"]; !ok {
		t.Error("trailer missing /Root")
	}
	if _, ok := trailer["Info"]; !ok {
		t.Error("trailer missing /Info")
	}

	rootRef, ok := trailer["Root"].(domain.PdfRef)
	if !ok || rootRef.ID != 1 {
		t.Errorf("trailer /Root: want PdfRef{1,0} got %v", trailer["Root"])
	}
	infoRef, ok := trailer["Info"].(domain.PdfRef)
	if !ok || infoRef.ID != 4 {
		t.Errorf("trailer /Info: want PdfRef{4,0} got %v", trailer["Info"])
	}
}
```

- [ ] **Step 3: Run test — verify it fails**

```
go test ./infra/... -run TestNewReader
```
Expected: compile error (NewReader not implemented).

- [ ] **Step 4: Implement xref loader + NewReader**

Replace the Reader stub at the bottom of `infra/reader.go` with:

```go
// ─── Reader ───────────────────────────────────────────────────────────────────

// Reader implements domain.ObjectResolver by seeking to object offsets on demand.
type Reader struct {
	f *os.File
}

// NewReader opens path, parses the xref table and trailer, and returns a ready Reader.
func NewReader(path string) (*Reader, *domain.XRefTable, domain.PdfDict, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, nil, nil, err
	}
	r := &Reader{f: f}
	xref, trailer, err := r.loadXRef()
	if err != nil {
		_ = f.Close()
		return nil, nil, nil, fmt.Errorf("loading xref from %q: %w", path, err)
	}
	return r, xref, trailer, nil
}

func (r *Reader) Close() error { return r.f.Close() }

// findStartXRef reads the last 1024 bytes of the file and returns the xref offset
// recorded after the "startxref" keyword.
func (r *Reader) findStartXRef() (int64, error) {
	size, err := r.f.Seek(0, io.SeekEnd)
	if err != nil {
		return 0, err
	}
	searchSize := int64(1024)
	if size < searchSize {
		searchSize = size
	}
	_, err = r.f.Seek(size-searchSize, io.SeekStart)
	if err != nil {
		return 0, err
	}
	buf := make([]byte, searchSize)
	if _, err := io.ReadFull(r.f, buf); err != nil {
		return 0, err
	}
	idx := bytes.LastIndex(buf, []byte("startxref"))
	if idx < 0 {
		return 0, fmt.Errorf("startxref keyword not found in last %d bytes", searchSize)
	}
	after := bytes.TrimLeft(buf[idx+len("startxref"):], " \t\r\n")
	end := 0
	for end < len(after) && after[end] >= '0' && after[end] <= '9' {
		end++
	}
	if end == 0 {
		return 0, fmt.Errorf("no integer after startxref")
	}
	return strconv.ParseInt(string(after[:end]), 10, 64)
}

// loadXRef seeks to the xref table and parses all subsections + the trailer dict.
func (r *Reader) loadXRef() (*domain.XRefTable, domain.PdfDict, error) {
	xrefOffset, err := r.findStartXRef()
	if err != nil {
		return nil, nil, err
	}
	if _, err := r.f.Seek(xrefOffset, io.SeekStart); err != nil {
		return nil, nil, err
	}

	p := newParser(r.f)
	xref := domain.NewXRefTable()

	// Expect "xref" keyword
	tok, err := p.tok.next()
	if err != nil || tok.kind != tokKeyword || string(tok.raw) != "xref" {
		return nil, nil, fmt.Errorf("expected 'xref' keyword at offset %d, got %q", xrefOffset, tok.raw)
	}

	// Parse subsections until "trailer"
	for {
		tok, err := p.tok.next()
		if err != nil {
			return nil, nil, err
		}
		if tok.kind == tokKeyword && string(tok.raw) == "trailer" {
			break
		}
		if tok.kind != tokInt {
			return nil, nil, fmt.Errorf("expected xref subsection header or 'trailer', got %q", tok.raw)
		}
		firstID, _ := strconv.ParseUint(string(tok.raw), 10, 32)

		countTok, err := p.tok.next()
		if err != nil || countTok.kind != tokInt {
			return nil, nil, fmt.Errorf("expected xref subsection count")
		}
		count, _ := strconv.ParseUint(string(countTok.raw), 10, 32)

		for i := uint64(0); i < count; i++ {
			t1, err := p.tok.next()
			if err != nil || t1.kind != tokInt {
				return nil, nil, fmt.Errorf("expected xref entry offset")
			}
			entryOffset, _ := strconv.ParseInt(string(t1.raw), 10, 64)

			t2, err := p.tok.next()
			if err != nil || t2.kind != tokInt {
				return nil, nil, fmt.Errorf("expected xref entry generation")
			}
			gen, _ := strconv.ParseUint(string(t2.raw), 10, 16)

			t3, err := p.tok.next()
			if err != nil || t3.kind != tokKeyword {
				return nil, nil, fmt.Errorf("expected xref entry type 'n' or 'f'")
			}

			objID := uint32(firstID) + uint32(i)
			if objID == 0 || string(t3.raw) != "n" {
				continue // skip free list head and free entries
			}
			xref.Insert(domain.XRefEntry{
				ObjID:  domain.ObjectID{ID: objID, Gen: uint16(gen)},
				Offset: entryOffset,
				InUse:  true,
			})
		}
	}

	// Parse trailer dict (we already consumed the "trailer" keyword above)
	trailerObj, err := p.parseObject()
	if err != nil {
		return nil, nil, fmt.Errorf("parsing trailer dict: %w", err)
	}
	trailer, ok := trailerObj.(domain.PdfDict)
	if !ok {
		return nil, nil, fmt.Errorf("trailer is not a dict")
	}

	return xref, trailer, nil
}
```

- [ ] **Step 5: Run test — verify it passes**

```
go test ./infra/... -run TestNewReader
```
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add infra/reader.go infra/reader_test.go
git commit -m "feat: add xref loader and NewReader"
```

---

## Task 9: ReadObjectAt

**Files:**
- Modify: `infra/reader.go` (add ReadObjectAt method)
- Modify: `infra/reader_test.go` (add ReadObjectAt integration test)

- [ ] **Step 1: Add the failing test**

Append to `infra/reader_test.go`:

```go
func TestReadObjectAt(t *testing.T) {
	r, xref, _, err := NewReader("../testdata/simple.pdf")
	if err != nil {
		t.Fatalf("NewReader: %v", err)
	}
	defer r.Close()

	// Object 1 = Catalog
	entry, ok := xref.Lookup(1)
	if !ok {
		t.Fatal("object 1 not in xref")
	}
	obj, err := r.ReadObjectAt(entry.Offset)
	if err != nil {
		t.Fatalf("ReadObjectAt(object 1): %v", err)
	}
	d, ok := obj.(domain.PdfDict)
	if !ok {
		t.Fatalf("object 1: want PdfDict got %T", obj)
	}
	if d["Type"] != domain.PdfName("Catalog") {
		t.Errorf("object 1 /Type: want Catalog got %v", d["Type"])
	}

	// Object 4 = Info
	entry4, ok := xref.Lookup(4)
	if !ok {
		t.Fatal("object 4 not in xref")
	}
	obj4, err := r.ReadObjectAt(entry4.Offset)
	if err != nil {
		t.Fatalf("ReadObjectAt(object 4): %v", err)
	}
	info, ok := obj4.(domain.PdfDict)
	if !ok {
		t.Fatalf("object 4: want PdfDict got %T", obj4)
	}
	if info["Title"] != domain.PdfString("Simple Test") {
		t.Errorf("Info /Title: want %q got %v", "Simple Test", info["Title"])
	}
	if info["Author"] != domain.PdfString("Test Author") {
		t.Errorf("Info /Author: want %q got %v", "Test Author", info["Author"])
	}
}
```

- [ ] **Step 2: Run test — verify it fails**

```
go test ./infra/... -run TestReadObjectAt
```
Expected: compile error (ReadObjectAt not defined).

- [ ] **Step 3: Implement ReadObjectAt**

Append to `infra/reader.go` (inside the Reader section, after `Close`):

```go
// ReadObjectAt implements domain.ObjectResolver.
// It seeks to offset, parses the indirect object header "n g obj … endobj",
// and returns the inner PdfObject value.
func (r *Reader) ReadObjectAt(offset int64) (domain.PdfObject, error) {
	if _, err := r.f.Seek(offset, io.SeekStart); err != nil {
		return nil, fmt.Errorf("seek to offset %d: %w", offset, err)
	}
	p := newParser(r.f)
	_, obj, err := p.parseIndirectObject()
	return obj, err
}
```

- [ ] **Step 4: Run all infra tests**

```
go test ./infra/...
```
Expected: PASS (all tokenizer, parser, reader tests)

- [ ] **Step 5: Commit**

```bash
git add infra/reader.go infra/reader_test.go
git commit -m "feat: implement ReadObjectAt — completes infra.Reader"
```

---

## Task 10: Application Editor

**Files:**
- Create: `application/editor.go`
- Create: `application/editor_test.go`

- [ ] **Step 1: Write the failing test**

```go
// application/editor_test.go
package application

import (
	"testing"
)

func TestOpen(t *testing.T) {
	ed, err := Open("../testdata/simple.pdf")
	if err != nil {
		t.Fatalf("Open: %v", err)
	}
	defer ed.Close()

	if n := ed.PageCount(); n != 1 {
		t.Errorf("PageCount: want 1 got %d", n)
	}
	meta := ed.Metadata()
	if meta["Title"] != "Simple Test" {
		t.Errorf("Title: want %q got %q", "Simple Test", meta["Title"])
	}
	if meta["Author"] != "Test Author" {
		t.Errorf("Author: want %q got %q", "Test Author", meta["Author"])
	}
}
```

- [ ] **Step 2: Run test — verify it fails**

```
go test ./application/... -run TestOpen
```
Expected: compile error.

- [ ] **Step 3: Implement**

```go
// application/editor.go
package application

import (
	"fmt"

	"github.com/felipemalacarne/papyrus/domain"
	"github.com/felipemalacarne/papyrus/infra"
)

// Editor is the primary handle for working with a PDF document.
type Editor struct {
	doc    *domain.Document
	reader *infra.Reader
}

// Open opens an existing PDF file and returns an Editor ready for reading.
func Open(path string) (*Editor, error) {
	reader, xref, trailer, err := infra.NewReader(path)
	if err != nil {
		return nil, fmt.Errorf("opening %q: %w", path, err)
	}

	doc := domain.NewDocument(xref, reader)

	rootRef, ok := trailer["Root"]
	if !ok {
		_ = reader.Close()
		return nil, fmt.Errorf("PDF trailer missing /Root")
	}
	root, ok := rootRef.(domain.PdfRef)
	if !ok {
		_ = reader.Close()
		return nil, fmt.Errorf("trailer /Root is not a reference")
	}
	doc.RootID = root

	if infoRef, ok := trailer["Info"]; ok {
		if info, ok := infoRef.(domain.PdfRef); ok {
			doc.InfoID = info
		}
	}

	return &Editor{doc: doc, reader: reader}, nil
}

// Metadata returns string values from the PDF Info dictionary.
func (e *Editor) Metadata() map[string]string {
	return e.doc.Metadata()
}

// PageCount returns the total number of pages in the document.
func (e *Editor) PageCount() int {
	return e.doc.PageCount()
}

// Close releases the underlying file handle.
func (e *Editor) Close() error {
	return e.reader.Close()
}
```

- [ ] **Step 4: Run test — verify it passes**

```
go test ./application/... -run TestOpen
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add application/editor.go application/editor_test.go
git commit -m "feat: add application.Editor with Open, Metadata, PageCount"
```

---

## Task 11: Public API + end-to-end test

**Files:**
- Create: `pdf.go`
- Create: `pdf_test.go`

- [ ] **Step 1: Write the failing end-to-end test**

```go
// pdf_test.go
package papyrus_test

import (
	"testing"

	papyrus "github.com/felipemalacarne/papyrus"
)

func TestOpenEndToEnd(t *testing.T) {
	ed, err := papyrus.Open("testdata/simple.pdf")
	if err != nil {
		t.Fatalf("papyrus.Open: %v", err)
	}
	defer ed.Close()

	if n := ed.PageCount(); n != 1 {
		t.Errorf("PageCount: want 1 got %d", n)
	}

	meta := ed.Metadata()
	if meta["Title"] != "Simple Test" {
		t.Errorf("Title: want %q got %q", "Simple Test", meta["Title"])
	}
	if meta["Author"] != "Test Author" {
		t.Errorf("Author: want %q got %q", "Test Author", meta["Author"])
	}
}
```

- [ ] **Step 2: Run test — verify it fails**

```
go test -run TestOpenEndToEnd .
```
Expected: compile error (papyrus.Open not defined).

- [ ] **Step 3: Implement**

```go
// pdf.go
package papyrus

import "github.com/felipemalacarne/papyrus/application"

// Open opens an existing PDF file for reading and returns an Editor.
// The caller must call Close() when done.
func Open(path string) (*application.Editor, error) {
	return application.Open(path)
}
```

- [ ] **Step 4: Run the full test suite**

```
go test ./...
```
Expected: PASS — all domain, infra, application, and root package tests.

- [ ] **Step 5: Commit**

```bash
git add pdf.go pdf_test.go
git commit -m "feat: expose public Open API — vertical slice complete"
```
