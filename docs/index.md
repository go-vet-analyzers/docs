# go-vet-analyzers

Pure-Go [`go vet`](https://pkg.go.dev/cmd/vet) analyzers — small, dependency-light
[`golang.org/x/tools/go/analysis`](https://pkg.go.dev/golang.org/x/tools/go/analysis)
tools that turn project conventions into checks that **fail CI** instead of
relying on human discipline.

Each analyzer ships as a `singlechecker` binary you drop into any repo's workflow:

```sh
go install github.com/go-vet-analyzers/<analyzer>/cmd/<analyzer>@latest
go vet -vettool="$(go env GOPATH)/bin/<analyzer>" ./...
```

## Analyzers

| Analyzer | What it checks |
| --- | --- |
| [`nonnil`](analyzers/nonnil.md) | Enforces the **Null-Object invariant** — fails the build when an interface declaring `IsNull() bool` is returned, assigned, or stored as a bare `nil`. |
| [`respondto`](analyzers/respondto.md) | Makes **reflective method-name strings compile-checked** — flags a `RespondTo("Method")` whose literal names a method the resolved type does not have. |

Both analyzers are deliberately conservative: they report only when they can be
certain, so false positives stay near zero. No cgo, 100% test coverage,
multi-OS/arch CI.

## License

BSD-3-Clause © the go-vet-analyzers authors.
