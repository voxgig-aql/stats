# Three-way test check: interpreter · check · byte compiler

This library's `.aql` suites are written once and must mean the same thing
no matter how `aql` runs them. `run.sh` runs every suite through all three
execution surfaces and asserts none errors or disagrees:

```bash
aql X            # interpreter — the default; what CI and users run
aql check X      # static type-check — must report 0 errors
aql --compile X  # byte compiler — bytecode when compilable, else a SILENT
                 #   fallback to the interpreter; documented to be IDENTICAL
                 #   to it ("opt-in performance, never semantics")
```

It also prints an `aql --force-compile X` coverage line per suite — how much
of each program the bytecode emitter can fully lower today. Refusals there
are expected gaps (under `--compile` they fall back to the interpreter), not
failures.

## Running it

```bash
test/divergence/run.sh
```

`run.sh` builds its own aql at a ref pinned in the script (the same
`12a44e0` the library pins; pinning it here keeps the harness
self-contained, so it never depends on whatever aql is on `PATH`), then
prints a per-suite matrix:

```
  SUITE                         INTERPRETER   CHECK           BYTECODE
  stats_unit_test.aql           ok            ok              ok
  stats_unit_spec.aql           ok            ok              ok
  stats_prop_test.aql           ok            ok              ok
  stats_prop_spec.aql           ok            ok              ok
  stats_smoke_test.aql          ok            ok              ok
```

It exits non-zero on any interpreter failure, any check **error**, or any
difference between `aql --compile X` and `aql X`. It fetches the aql source
as a codeload tarball (curl), so it builds even where raw `git clone` of
aql-lang/aql is blocked; needs `go` + network for the one-time build (cached
in `~/.cache/aql-divergence`).

## Background: what this guards against

`aql --compile` is documented to return results identical to the interpreter
(it falls back to the interpreter for anything it can't lower). This harness
exists because that promise has been broken before — notably a compiled
`each` body once dropped a *block-local* binding from its enclosing block,
and because the emitter believed it could lower the body, `--compile` did
**not** fall back and a wrong result escaped.

That class of bug matters here: several words in `stats.aql` bind a local
inside an `each` body and read it there (e.g. `centered-rows` binds `def row
(mat MatrixUtil.row i)` per row; `cor-matrix` binds `def crow …`). On this
pin (`12a44e0`) those compile byte-identically to the interpreter, and the
harness asserts it on every change so a future regression can't slip the
"identical, never semantics" guarantee past CI.

`--force-compile` fully compiles `stats_prop_test.aql`; the others refuse on
code-body words (`each` / `test-test`, "Stage 2") or an `fn` operand of
unknown provenance, and fall back cleanly under `--compile` — sound by
`aql-lang/aql`'s `design/COMPILABLE-SUBSET.md` ("refusal is always sound;
the worst failure mode is slow, not wrong").

### Wiring it into CI

`run.sh` is self-contained, so a gating job is one block (add it to
`.github/workflows/test.yml` — needs a token with `workflow` scope, which the
agent session that wrote this didn't have):

```yaml
  divergence:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
      - name: interpreter / check / byte-compiler agreement
        run: test/divergence/run.sh
```
