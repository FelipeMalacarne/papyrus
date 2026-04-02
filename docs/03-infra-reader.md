# Phase 3 — Infra: Reader

All I/O lives here. Reader implements `domain.ObjectResolver`.

## Step 7 — Tokenizer (`infra/reader.go`)

```go
package infra

type TokenType int

const (
    TokenName TokenType = iota
    TokenString
    TokenInt
    TokenReal
    TokenBool
    TokenNull
    TokenRef       // "12 0 R"
    TokenDictOpen  // "<<"
    TokenDictClose // ">>"
    TokenArrayOpen
    TokenArrayClose
    TokenStreamBegin
    TokenKeyword   // obj, endobj, xref, trailer, startxref
)

type Token struct {
    Type  TokenType
    Raw   []byte
}

type Tokenizer struct {
    r   *bufio.Reader
    pos int64
}

func NewTokenizer(r io.ReadSeeker) *Tokenizer
func (t *Tokenizer) Next() (Token, error)
func (t *Tokenizer) Peek() (Token, error)
```

### Test checklist

- [ ] Tokenize `<< /Type /Page >>` → DictOpen, Name, Name, DictClose
- [ ] Tokenize `12 0 R` → TokenRef {ID:12, Gen:0}
- [ ] Tokenize integer, real, bool, null
- [ ] Tokenize literal string `(Hello\nWorld)`
- [ ] Tokenize hex string `<48656c6c6f>`

---

## Step 8 — Object parser (`infra/reader.go`)

```go
func parseObject(t *Tokenizer) (domain.PdfObject, error)
// handles: dict, array, stream (reads Length bytes), all scalar types
// streams: reads raw bytes, does NOT decode (decoder.go handles that)
```

### Test checklist

- [ ] Parse nested dict with array value
- [ ] Parse stream: assert Dict keys present and Data length matches /Length
- [ ] Parse indirect object header `12 0 obj ... endobj`
- [ ] Return error on malformed input

---

## Step 9 — XRef loader (`infra/reader.go`)

```go
type Reader struct {
    f        *os.File
    size     int64
    tokenizer *Tokenizer
}

func OpenReader(path string) (*Reader, error)
func (r *Reader) Close() error

// LoadXRef: seek to startxref offset, parse xref table + trailer
// Returns populated XRefTable, root ObjectID, info ObjectID
func (r *Reader) LoadXRef() (*domain.XRefTable, domain.ObjectID, domain.ObjectID, error)
```

Classic xref table format:
```
xref
0 6
0000000000 65535 f
0000000009 00000 n
...
trailer
<< /Size 6 /Root 1 0 R /Info 2 0 R >>
startxref
491
%%EOF
```

### Test checklist

- [ ] Open a 1-page PDF fixture, assert xref entry count
- [ ] Assert RootID and InfoID are non-zero
- [ ] Assert specific obj offsets match known fixture values

---

## Step 10 — ReadObjectAt (`infra/reader.go`)

Implements `domain.ObjectResolver`.

```go
func (r *Reader) ReadObjectAt(offset int64) (domain.PdfObject, error)
// 1. Seek to offset
// 2. Parse "N G obj" header
// 3. Call parseObject()
// 4. Parse "endobj"
// 5. If stream: check /Filter, decode via decoder.go
```

### Test checklist

- [ ] ReadObjectAt on known offset returns expected PdfDict keys
- [ ] ReadObjectAt on stream object returns decoded Data (FlateDecode fixture)
- [ ] Reader satisfies domain.ObjectResolver interface (compile-time check)

---

## Decoder (`infra/decoder.go`)

```go
func Decode(filter domain.PdfName, data []byte) ([]byte, error)
// supported: FlateDecode (zlib), ASCIIHexDecode
// returns error for unsupported filters
```
