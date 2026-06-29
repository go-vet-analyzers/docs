# nonnil — never return a bare `nil` for a Null-Object interface



`github.com/go-vet-analyzers/nonnil` is a
[`golang.org/x/tools/go/analysis`](https://pkg.go.dev/golang.org/x/tools/go/analysis)
analyzer (a `go vet` tool) that enforces the **Null-Object invariant**: a
value whose type is a *Null-Object interface* — any interface that declares
`IsNull() bool` — must never be a bare `nil`. It must instead be a real Null
object (`Null.New()`, [`NullError.New()`](https://go-composites.github.io/docs/components/error/), …) that
callers can always send messages to.

This turns "always return a Null object, never `nil`" from a matter of human
discipline into a check that **fails CI**, closing the one gap that lets the
nil-dereference the pattern exists to prevent slip back in.

## What it enforces

The analyzer treats an interface type as a **Null-Object interface** when that
type declares an `IsNull() bool` method — taking no parameters and returning a
single `bool` — directly or through embedding. (`isNullObjectInterface` resolves
this with `types.LookupFieldOrMethod`.)

At every site where a bare `nil` would re-enter such a type, it reports a
diagnostic:

- **`return nil`** — a `nil` in any result position of a function or function
  literal whose declared result type is a Null-Object interface. Only the
  position-aligned form (`return a, b, c`) is inspected; a naked return or a
  single call feeding multiple results (`return f()`) is left alone to avoid
  false positives.
- **`x = nil`** — assignment of `nil` to a Null-Object-typed variable or field.
- **`var x T = nil`** — initialisation of a value spec whose explicit type is a
  Null-Object interface.
- **composite literals** — a `nil` element targeting a Null-Object-typed slot:
  a struct field (`T{Field: nil}` or positional), a map value
  (`map[K]T{k: nil}`), or a slice/array element (`[]T{nil}`, `[N]T{nil}`).

The diagnostic for a return reads:

```
return a Null object (e.g. Null.New()) instead of nil for <Type>: it is a
Null-Object interface (IsNull() bool), and nil reintroduces the nil-dereference
the pattern prevents
```

Assignment / construction sites produce the parallel message
`assign`/`initialise`/`set field … to`/`set map value to`/`set element to` a
Null object instead of `nil`.

!!! note "Deliberately conservative"
    Interfaces **without** `IsNull()` are never flagged — including the builtin
    `error` and `Result.Interface` — so false positives are minimal. A typed
    conversion `return (Thing)(nil)`, a nil interface *variable*, an omitted
    struct field (its zero value), and a nil map *key* are all left alone, since
    they are explicit, structural, or uncommon.

## Flagged vs. accepted

```go
import (
    Error     "github.com/go-composites/error/src"
    NullError "github.com/go-composites/error/src/null"
)

// FLAGGED: Error.Interface declares IsNull() bool, and this returns a bare nil.
func lookup(bad bool) Error.Interface {
    if bad {
        return nil // nonnil: return a Null object (e.g. Null.New()) instead of nil
    }
    return Error.New("boom")
}

// ACCEPTED: hands back a real Null object the caller can always message.
func lookupOK(bad bool) Error.Interface {
    if bad {
        return NullError.New()
    }
    return Error.New("boom")
}
```

```go
type Box struct {
    Err Error.Interface
}

_ = Box{Err: nil}            // FLAGGED: set field Err to a Null object instead of nil
_ = Box{Err: NullError.New()} // ACCEPTED

var e Error.Interface = nil  // FLAGGED: initialise a Null object instead of nil
e = nil                      // FLAGGED: assign a Null object instead of nil

var ok error = nil           // ACCEPTED: builtin error has no IsNull(), not a Null-Object interface
```

## Install & run

```sh
go install github.com/go-vet-analyzers/nonnil/cmd/nonnil@latest
go vet -vettool=$(which nonnil) ./...
```

The command wraps the analyzer with `singlechecker`, so it also runs standalone
and accepts the usual `go vet` flags and package patterns.

## CI integration

Every org repo runs `nonnil` on push and pull request via the `nonnil.yml`
workflow, which installs the tool and vets the whole module:

```yaml
  - run: go install github.com/go-vet-analyzers/nonnil/cmd/nonnil@latest
  - run: go vet -vettool="$(go env GOPATH)/bin/nonnil" ./...
```

A bare `nil` in any of the positions above turns the job red, making the mistake
un-mergeable.

## License

BSD-3-Clause © the go-vet-analyzers/nonnil authors.
