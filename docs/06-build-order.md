# Build order + dependency map

Each step only depends on steps above it. Steps in the same phase can be developed in parallel.

```
Phase 1 (no deps)
  1. PdfObject types
  2. XRefTable
  3. DomainEvent types

Phase 2 (needs Phase 1)
  4. Document + ObjectResolver interface
  5. NodeRef[T]
  6. Node types (Catalog, PageTree, Page, ContentStream, ResourceDict)

Phase 3 (needs Phase 1 only — parallel with Phase 2)
  7. Tokenizer
  8. Object parser          (needs 7)
  9. XRef loader            (needs 7, 8)
  10. ReadObjectAt           (needs 7, 8, 9 — wires into domain.ObjectResolver)

Phase 4 (needs Phase 2 + Phase 3)
  11. StreamOp types + compiler
  12. Box, Text, Image components   (needs 11)
  13. Page component tree           (needs 12, needs Page from step 6)

Phase 5 (needs all above)
  14a. Serializer primitives
  14b. SaveIncremental              (needs 14a)
  14c. Save full rewrite            (needs 14a)
  15.  Application layer            (needs 14b, 14c, all domain + infra)
  16.  Public API + e2e tests       (needs 15)
```

## Parallel tracks

Once Phase 1 is done you can split:
- **Track A**: Phase 2 (domain) → Phase 4 (components) — no file I/O needed, pure unit tests
- **Track B**: Phase 3 (reader) — all the parsing work, integration tests with real PDF fixtures

Both tracks converge at Phase 5.

## Test fixture PDFs

Keep a `testdata/` directory with:
- `simple.pdf` — 1 page, no images, classic xref table, no encryption
- `multipage.pdf` — 3+ pages for page tree tests
- `with-images.pdf` — for XObject / ResourceDict tests
- `compressed-stream.pdf` — FlateDecode stream for decoder tests

Generate fixtures with any trusted tool (LibreOffice, Ghostscript) so they are known-good baselines.
