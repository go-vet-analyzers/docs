# respondto — make reflective method-name strings compile-checked



`github.com/go-vet-analyzers/respondto` is a
[`golang.org/x/tools/go/analysis`](https://pkg.go.dev/golang.org/x/tools/go/analysis)
analyzer (a `go vet` tool) that catches typo'd or stale **reflective method
sends** at build time.

In the composition style, an object is asked whether it implements a method *by
name*, using a string and reflection:

```go
Object.RespondTo(instance, "Message") // package-level helper
instance.RespondTo("Message")         // method form
```

Because the name lives in a plain string the compiler never inspects, a typo or
a renamed method compiles fine and only blows up at runtime. `respondto`
resolves the receiver/object's type and flags a string-literal method name that
is **not in that type's method set** — turning "the string must match a real
method" into a check that **fails CI**.

## What it checks

The analyzer walks every call expression and reports **only** when *all* of the
following hold (it never guesses):

- **The callee is named exactly `RespondTo`**, in one of three syntactic shapes:
    - `RespondTo(object, "Method")` — an unqualified package-level function; the
      object is the first argument.
    - `pkg.RespondTo(object, "Method")` — a qualified package-level function; the
      object is the first argument.
    - `x.RespondTo("Method")` — the method form; the receiver is the selector
      base `x`.

    For the package-level shapes the callee must resolve to an actual
    package-scoped `*types.Func` (not a local closure that happens to be named
    `RespondTo`).
- **The method-name argument is a string constant** — a literal, or a `const`
  that folds to a string (e.g. `"He"+"llo"`). A variable, field, or any other
  computed/dynamic expression is ignored.
- **The receiver/object type has a determinable method set** — a named type
  (struct or defined), a pointer to one, or a non-empty interface. An empty
  interface (`interface{}`/`any`), a type parameter, or an unnamed non-interface
  type yields no judgement and is skipped.
- **That type has no method of the given name**, looked up across the full
  method set including embedded/promoted methods via
  `types.LookupFieldOrMethod`.

When a name fails the lookup, it reports:

```
RespondTo: <Type> has no method "<Name>" (stale or misspelled method name)
```

!!! note "Pointer and embedding awareness"
    The receiver type is kept as-is (a single pointer is unwrapped only to read
    the type name), so `*T` value- and pointer-methods both resolve. Methods
    promoted from embedded interfaces or structs count as present.

## Flagged vs. accepted

```go
type Greeter interface {
    Hello() string
    Bye() string
}

func packageLevel(g Greeter) {
    _ = RespondTo(g, "Hello") // ACCEPTED: Greeter has Hello
    _ = RespondTo(g, "Nope")  // FLAGGED: RespondTo: Greeter has no method "Nope"
}

func methodForm(w widget) {
    _ = w.RespondTo("Hello")  // ACCEPTED: widget has Hello
    _ = w.RespondTo("Typo")   // FLAGGED: RespondTo: widget has no method "Typo"
}
```

Promoted methods are recognised; dynamic and unresolvable cases are left alone:

```go
type Base interface{ Ping() string }
type Derived interface {
    Base
    Bar() int
}

_ = RespondTo(d, "Ping")          // ACCEPTED: Ping is promoted from embedded Base

_ = RespondTo(g, name)            // ACCEPTED: non-literal argument, ignored
var any interface{} = g
_ = RespondTo(any, "Whatever")    // ACCEPTED: empty interface, no method set to judge
var p *int
_ = RespondTo(p, "Nope")          // ACCEPTED: pointer to unnamed type, no method set
```

!!! tip "The failure it prevents"
    Rename a method, miss one `RespondTo("…")` call site, and the program keeps
    building — only to fail (or silently mis-dispatch) at runtime. `respondto`
    surfaces that stale string as a build error instead.

## Install & run

```sh
go install github.com/go-vet-analyzers/respondto/cmd/respondto@latest
go vet -vettool=$(which respondto) ./...
```

The command wraps the analyzer with `singlechecker`, so it also runs standalone
and accepts the usual `go vet` flags and package patterns.

## CI integration

Run `respondto` on every org repo by adding it to the workflow:

```yaml
  - run: go install github.com/go-vet-analyzers/respondto/cmd/respondto@latest
  - run: go vet -vettool="$(go env GOPATH)/bin/respondto" ./...
```

A string-literal method name that does not exist on its resolved type turns the
job red, so a typo'd or renamed dynamic send can never merge.

## License

BSD-3-Clause © the go-vet-analyzers/respondto authors.
