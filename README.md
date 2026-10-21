# symbolindex

`symbolindex` is a small Go library for turning source text into a storage-agnostic symbol graph.

It accepts caller-provided file contents, parses the supported languages, and returns structured metadata for symbols, edges, references, and calls. It does not discover files on disk or manage persistence itself.

---

## What it does

`symbolindex` is focused on symbol-level analysis. Given source files like this:

```go
file := symbolindex.SourceFile{
    Path:     "example.go",
    Content:  "package demo\n\nfunc Hello(name string) string { return \"hi \" + name }\n",
    Language: symbolindex.LanguageGo,
}
```

it can index:

- package and namespace declarations
- types, methods, functions, fields, properties, and events
- symbol relationships such as `contains`, `declares`, `references`, `calls`, `inherits`, `implements`, and `imports`
- call sites and candidate targets
- symbol usage locations and diagnostics for ignored or unsupported inputs

This is designed for downstream search, graph queries, indexing pipelines, and repository analysis without coupling to a database or filesystem layer.

---

## Supported languages

The current implementation supports:

- Go
- C#

Files with unsupported languages are skipped with a warning diagnostic. Test files and generated files can be filtered through `Options`.

---

## Installation

This repository is a Go module:

```bash
go get github.com/KaiserWerk/go-indexing-libraries/symbolindex
```

Then import it:

```go
import "github.com/KaiserWerk/go-indexing-libraries/symbolindex"
```

---

## Basic usage

```go
package main

import (
    "fmt"

    "github.com/KaiserWerk/go-indexing-libraries/symbolindex"
)

func main() {
    files := []symbolindex.SourceFile{
        {
            Path:     "example.go",
            Content:  "package demo\n\nfunc Hello(name string) string { return \"hi \" + name }\n",
            Language: symbolindex.LanguageGo,
        },
    }

    result, diagnostics, err := symbolindex.Index(files, symbolindex.Options{
        DefaultContextID: "demo",
    })
    if err != nil {
        panic(err)
    }

    fmt.Println("symbols:", len(result.Symbols))
    fmt.Println("edges:", len(result.Edges))
    fmt.Println("calls:", len(result.Calls))
    fmt.Printf("diagnostics: %#v\n", diagnostics)
}
```

---

## Input model

```go
type SourceFile struct {
    Path      string
    Content   string
    Language  Language
    ContextID string
}
```

`Path` and `Content` are required. `Language` can be set explicitly when needed; otherwise the extension is detected automatically.

`Options` supports:

```go
type Options struct {
    LanguageOverrides map[string]Language
    DefaultContextID  string
    IncludeTests      bool
    IncludeGenerated  bool
}
```

The default context ID is `"default"`.

---

## Output shape

`symbolindex.Index(...)` returns a `Result`:

```go
type Result struct {
    Symbols    []Symbol
    Edges      []Edge
    References []Reference
    Calls      []Call
}
```

Key types:

```go
type Symbol struct {
    ID                   string
    StableKey            string
    Language             Language
    Kind                 SymbolKind
    LanguageSpecificKind string
    Name                 string
    FQN                  string
    Signature            string
    ContainerFQN         string
    ContextID            string
    Declarations         []Span
}

type Edge struct {
    Type      EdgeType
    FromID    string
    ToID      string
    ContextID string
}

type Reference struct {
    FromID      string
    ToID        string
    ContextID   string
    Location    Span
    Candidates  []Candidate
    IsAmbiguous bool
}

type Call struct {
    CallerID    string
    CalleeID    string
    ContextID   string
    Location    Span
    Candidates  []Candidate
    IsAmbiguous bool
}
```

These records are intentionally storage-agnostic and easy to persist in your own application layer.

---

## Search summary output

`symbolindex` also includes a lightweight query-driven summary for CLI or debug output.

```go
summary, err := symbolindex.BuildSearchSummary("auth", files, result)
if err != nil {
    panic(err)
}

fmt.Println(symbolindex.FormatSearchSummary(summary))
```

This produces output shaped like:

```text
Search: "auth"

Relevant files:

1. internal/app/auth/service.go
   Symbols: Service, Service.AuthenticateRequest, Service.LoginWithPassword
   Text matches: 4

Relationships:
  DoPasswordLogin
    -> Service.LoginWithPassword
```

The summary includes:

- relevant files matching the query
- symbol names found in those files
- text match counts
- direct call relationships between relevant symbols

---

## Recommended usage pattern

Use the library as a pipeline:

1. read source files into memory
2. pass them as `SourceFile` values
3. call `symbolindex.Index(...)`
4. store or query the result in your own app or service
5. optionally build a human-readable summary with `BuildSearchSummary(...)`

This keeps indexing logic decoupled from repository traversal, persistence, and filesystem concerns.

---

## Best practices

- Prefer passing `Path` and `Content` explicitly for deterministic indexing
- Set `Language` when a filename is ambiguous or custom
- Use `ContextID` to separate repository or project scopes
- Filter generated/test files early using `Options`
- Persist the results in your own storage layer instead of embedding DB logic in the indexer

---

## Summary

`symbolindex` is intentionally small and focused: it answers the question, “What symbols and relationships exist in this source?”

It is designed to produce structured, readable metadata that is easy to store, query, and reason about in larger code intelligence or repository-analysis pipelines.