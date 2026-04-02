# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Papyrus** is a Go library for PDF reading, modifying, and creation. Module: `github.com/felipemalacarne/papyrus`. Go 1.24.3+. No external dependencies — only the standard library.

## Commands

```bash
go build ./...
go test ./...
go test ./... -run TestName          # single test
go vet ./...
```

## Architecture

The project follows a **DDD-lite layered architecture**. All design documentation is in `docs/`.

### Layers (strict dependency order)

```
domain/ → infra/ + components/ → application/ → pdf.go (public API)
```

**`domain/`** — Pure business logic, zero I/O.
- `PdfObject`: sum type (Dict, Array, Name, String, Int, Real, Bool, Null, Reference, Stream)
- `XRefTable`: in-memory index mapping object numbers to file offsets
- `Document`: aggregate root; caches resolved objects; fires `DomainEvent`s on mutation
- `NodeRef[T]`: lazy handle to any typed node — dereferences on first access via injected `ObjectResolver`
- `DomainEvent`: `ObjectModified | ObjectRemoved | PageAdded | PageRemoved`

**`infra/`** — All I/O.
- `Reader`: tokenizer → object parser → xref loader; implements the `ObjectResolver` interface consumed by `Document`
- `Decoder`: FlateDecode, ASCIIHexDecode
- `Serializer`: `Save()` (full rewrite) and `SaveIncremental()` (append-only, driven by domain events)

**`application/`**
- `Editor`: orchestrates `Open()` / `New()` / `Save()` / `SaveIncremental()`

**`components/`** — PDF drawing primitives.
- `StreamOp` types: graphics state, color, paths, transforms, text, XObjects
- `Component` interface with fluent builders: `Box`, `Text`, `Image`
- `Recompile()` renders the component tree into PDF content stream bytes

**`pdf.go`** — Thin public API surface.

### Key design decisions

- **Lazy loading**: `NodeRef[T]` defers I/O to first access; objects are never fully loaded upfront.
- **Dependency inversion**: `domain` never imports `infra`; `Reader` is injected as `ObjectResolver`.
- **Two save modes**: incremental append (replay domain events) vs. full rewrite (serialize entire object graph).
- **Phased build order**: Domain types → Document/NodeRef → Reader (parallel) → Components → Serializer/API. See `docs/06-build-order.md` for the dependency graph.

### Test fixtures

Place PDF test fixtures under `testdata/`: `simple.pdf`, `multipage.pdf`, `with-images.pdf`, `compressed-stream.pdf`. Each doc phase has a test checklist — see `docs/01-domain-objects.md` through `docs/05-serializer-api.md`.
