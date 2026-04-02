# papyrus — project overview

A Go library for reading, modifying, and creating PDF files. Designed around a DDD-lite architecture with a clear separation between domain, infrastructure, and application layers.

## Goals

- Lazy-load PDF objects on demand without reading the full file
- Modify, add, and remove components via a typed component tree
- Save changes as incremental updates (appending to existing file) or full rewrites (for new files)
- Expose a fluent, composable public API

## Non-goals (for now)

- PDF rendering / rasterization
- Font subsetting / embedding
- Encryption / decryption
- XRef stream parsing (PDF 1.5+ compressed xref) — classic table only initially

## Repository layout

```
papyrus/
├── pdf.go                  # public API surface
├── domain/
│   ├── object.go           # PdfObject types
│   ├── xref.go             # XRefTable
│   ├── events.go           # DomainEvent types
│   ├── document.go         # aggregate root
│   ├── noderef.go          # NodeRef[T] lazy handle
│   └── nodes/
│       ├── catalog.go
│       ├── page_tree.go
│       ├── page.go
│       ├── content_stream.go
│       └── resource_dict.go
├── infra/
│   ├── reader.go           # tokenizer + object parser + xref loader
│   ├── decoder.go          # FlateDecode etc.
│   └── serializer.go       # writeObject, SaveIncremental, Save
├── application/
│   └── editor.go           # Open(), New(), orchestration
└── components/
    ├── component.go         # Component interface + StreamOp types
    ├── box.go
    ├── text.go
    └── image.go
```

## Key design decisions

- `domain/` has zero I/O — no `os`, no `bufio`, no file handles
- `infra/Reader` implements `domain.ObjectResolver` interface — injected into Document
- `NodeRef[T]` is the lazy handle for every edge in the PDF object graph
- Domain events drive both save modes — serializer consumes `doc.PendingEvents()`
- `Save()` (full rewrite) and `SaveIncremental()` share primitive helpers in serializer
