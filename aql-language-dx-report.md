# A developer-experience report on the AQL language

**Date:** 2026-06-25
**Build under test:** `aql-lang/aql` @ `12a44e0`
(`12a44e0c6ca3f49cd35a871b573fd96bc13d7fd6`, main as of 2026-06-24, PR
#189; built locally with `GOFLAGS=-mod=mod`; `aql -version` reports
`aql 12a44e0-main`).
**Vantage point:** writing one non-trivial library from scratch — the
`Stats` statistics module in this repo: ~840 lines exporting one
namespace and a class, with five test suites (example + property,
imperative + declarative, plus a smoke run) that all pass across the
interpreter, `aql check`, and the byte compiler. This is a *consumer's*
view of the language, not a compiler-internals view.

This report is the language-level companion to [`dx-report.md`](dx-report.md),
which logs the specific runtime gotchas this module worked around. Here
I step back: what AQL gets right, where it bites, and what would most
improve the experience of writing real code in it.

Severity for the issues below: **🔴 high** (silent wrong results, crash,
or a blocked use case) · **🟡 medium** (friction with a clear
workaround) · **🟢 low** (papercut).

---

## Verdict in one paragraph

AQL is a structure-first, concatenative language with a genuinely
unusual and genuinely valuable property: the same source is run three
ways — interpreter, static checker, and byte compiler — and the
toolchain *guarantees* the compiler never changes a program's meaning
(it falls back to the interpreter rather than risk it). That contract is
the best thing about developing in AQL, and it caught real divergences
for this module. The cost is a calling convention with several sharp
edges that fail *silently* (wrong value, not an error) when you get them
slightly wrong, and a static checker that is not yet trustworthy enough
to gate on directly. None of these blocked the library; all of them cost
debugging time that a clearer failure mode would have saved.

---

## What works well

**The three-surface model is excellent.** `aql X` (interpret),
`aql check X` (static type-check), and `aql --compile X` (byte-compile,
falling back to the interpreter for anything it can't lower) are framed
as "opt-in performance, never semantics." `aql --force-compile X` tells
you *how much* of a program the emitter can lower today, and refusals
there are documented as always sound. Building a differential harness on
top of this (`test/divergence/run.sh`) was straightforward and it earns
its keep: it asserts `aql --compile X == aql X` on every suite, so a
compiler regression can't slip a wrong result past CI. Very few small
languages give you this guarantee, let alone a CLI flag to measure
coverage of it.

**The module system is clean.** `import "aql:math-util"` binds a single
namespace (`MathUtil`); a local library imports its own dependencies and
exports one namespace via an auto-evaluating map literal, with `/r` to
defer dispatch of the exported words. Re-export is *not* implicit, which
is the right default (callers don't accidentally inherit a transitive
binding) even though it surprised me once (a consumer building a `Matrix`
must import `aql:matrix-util` itself).

**The error model is good.** `raise code message` with template-literal
messages, caught by `do […] error […]` with `e get code` / `e get
message`, is ergonomic and reads well. Coding the four failure modes of
this library (`bad_input`, `needs_data`, `singular`, `bad_payload`) and
asserting them in tests was painless.

**The numeric surface is real.** `aql:math-util` carries a full
IEEE-754-aware vocabulary — `sqrt`/`log`/`exp`, the rounding family,
`fma`, `hypot`, `nextafter`, `copysign`, NaN/inf classifiers, and
order-independent NaN-ignoring `min`/`max`. Integer overflow is a *hard
error*, not a silent wrap, which is the safe choice and pushed me to
float-up sums of squares deliberately rather than discover a wrap in
production.

**The test framework is unusually complete for a small language.**
`aql:test` ships both an imperative surface (`Test.test`,
`Test.check-prop`) and a declarative one (`Test.run-spec`,
`Test.run-property`), property-based testing with shrinking, a frozen
clock for determinism, and result records you can iterate. I was able to
write example-based and property-based suites in both styles without
reaching outside the stdlib.

**Small niceties that add up:** `aql -version` stamps the git commit (so
"which build am I on?" answers itself); `MathUtil.floor`/`ceil` on a
Float return an `Integer`, usable directly as a list index; the
`describe`/docs surface is generated and complete.

---

## Friction and sharp edges

The through-line: AQL's worst failures are **silent**. Because the
calling convention is positional and token-collecting, a small mistake
often yields a *wrong value* or a `None` that fails three lines later,
rather than an error at the call site. Each of these cost real debugging
time.

### 1. 🔴 Per-call frame cleanup over-pops with repeated `comp/r apply`

The sharpest bug I hit is reproducible and looks like a frame-cleanup
over-pop: a user function that calls a higher-order helper via `comp/r`
**twice** loses an enclosing binding after the first call.

```aql
def g ([x:Integer] => [x add 1])
def h fn [[comp:Function v:Integer] [Integer] [v comp/r apply]]
def t fn [[comp:Function] [Integer] [
  def a (5 comp/r h)
  def b (7 comp/r h)        # <- errors here
  a add b
]]
print ((g/r t)) end          # expected 13
# => error: undefined word: comp   (at the SECOND call)
```

A single call works (`def t … [ def a (5 comp/r h) a ]` returns `6`);
the second call to a `comp/r`-using helper has popped `comp` out of
`t`'s frame. (This matches what `design/ACCESSOR-SPLIT-AND-CLEANUP-BUG.md`
in the aql repo describes.) It's loud *here*, but a frame-cleanup
over-pop is the kind of defect that can equally surface as a wrong value;
either way it makes higher-order code that re-invokes a captured function
unreliable. This is the one finding I'd fix first.

### 2. 🔴 `get`/`set` read a bare-word index as an atom key, silently

The index/key argument to `get`/`set` is taken literally, so a bare
*variable* is read as the atom of its name, not its value — and on a List
that yields `None`, not an error:

```aql
def s [10.0 20.0 30.0]
def i 1
s get 1     # => 20.0   (literal index works)
s get i     # => None   (read as the atom `i`)
s get (i)   # => 20.0   (parenthesise to evaluate)
```

The `None` then detonates in later arithmetic far from the cause. The
rule "parenthesise every variable index" is learnable, but the failure
should be a dispatch error at the `get`, not a silent `None`.

### 3. 🔴 A map-literal value `{k: [expr]}` only evaluates under `do`

The bracketed-value form evaluates only when the literal is prefixed
with `do`; a bare `{…}` stores the brackets as a literal List:

```aql
def v 23.0
def a {best: [v]}        # a.best == [23.0]   (a one-element List!)
def b (do {best: [v]})   # b.best == 23.0     (evaluated)
```

This produced a subtly wrong result in this module's `mode` word (a
seed map built without `do` carried a List where a scalar was meant),
caught only by a property test. The two spellings looking nearly
identical while meaning different things is the trap.

### 4. 🟡 Forward-argument order is surprising for native multi-arg words

Two `aql:matrix-util` words bind their forward operands in an order that
inverts the natural reading:

- `MatrixUtil.mat-mul A B` computes **B·A**, not A·B (verified by shape:
  a 2×3 times a 3×2 returns 3×3). The spec's examples are all square, so
  the order never shows there.
- `MatrixUtil.elem c r` is `(col, row)`, not `(row, col)` (this one *is*
  documented, but it inverts the usual convention).

A wrong order is a silent wrong-shape / wrong-cell result. Both the
covariance matrix and OLS in this module were wrong until I pinned the
order down against known answers. Consistent left-to-right operand
binding (or at least a loud shape error) would remove a whole class of
mistake.

### 5. 🟡 `aql check` is not yet trustworthy enough to gate on

The checker reports *hard errors* for code that runs correctly,
especially around gradual `Any`. A query word typed `[x:Any]` that
dispatches `x` to a `List`-typed helper draws `no_signature: no matching
signature …` even though `Any` should accept it; worse, the report flips
depending on which arm of a union the value provably is. The shape that
finally type-checked clean in both directions needed a union parameter
*and* narrowing on the **positive** branch:

```aql
def as-summary fn [
  [x:(List tor Summary)] [Summary] [
    if (x is List) [build-summary x] [x]   # `is List`; `is Summary`-negative still errored
  ]
]
```

There's also a typing quirk where an empty-list literal `[]` flowing
into such a word poisons the inferred type at top level (but not inside
an opaque `Test.test`/`each` body). The net effect: `aql check` on a
library *in isolation* still surfaces false `unused_def` (export-by-
reference hides use sites) and `no_signature` findings, so it can't be a
gate by itself. The workable pattern — which this repo's CI uses — is to
gate on `aql check` *through the test suites* (where words are called
with concrete types) and treat the standalone check as advisory. That
works, but "the static checker is advisory" is a tax on a language whose
other surfaces are so disciplined.

### 6. 🟡 No tagged release; `go install` doesn't work

AQL has no tagged release, and the documented `go install
…/cmd/go/aql@latest` can't run while `cmd/go/go.mod` carries replace
directives. Every consumer must build from a pinned commit. That's
workable (this repo pins one commit across the hook, CI, and the
divergence harness, with a consistency job to stop drift), but it means
"add this library to my project" starts with "build the interpreter from
source at exactly this SHA." A tagged release with a `go install`-able
entrypoint, or prebuilt binaries, would dramatically lower the barrier.

### 7. 🟢 Smaller papercuts

- `print` collects a forward argument, so chained `(a) print (b) print`
  prints out of order; the safe idiom is one `print (value) end` per
  statement.
- A one-letter uppercase identifier (`def M …`) is parsed as a type
  variable and rejected as a value binding; values must be lowercase.
- `StructUtil.parse` collapses a whole-valued Float like `42.0` to
  Integer `42`, and a sealed `class` field then rejects it on `make`, so
  decode paths must coerce field types back.
- Direct `print (aList)` renders a List comma-separated (`[4.0, 7.5]`)
  while `${aList}` interpolation renders it space-separated
  (`[4.0 7.5]`) — easy to trip on when writing expected-output comments.

---

## Tooling and build experience

- **Build:** `GOFLAGS=-mod=mod go build ./aql` from `cmd/go` is reliable.
  Fetching the source as a codeload tarball (`curl … | tar -xz`) is more
  robust than `git clone` in locked-down sandboxes that allow the API/
  codeload hosts but block the raw git host — both this repo's
  SessionStart hook and its divergence harness fetch that way.
- **Diagnostics:** runtime errors carry a source span and a *remarkably*
  helpful hint — e.g. "forward args for `convert` may have run into the
  next word; group the call in parens" — which repeatedly pointed me at
  the real fix. This is one of the best parts of the day-to-day loop and
  partly compensates for the silent-failure modes above (once a wrong
  value finally does error, the message is usually actionable).
- **Determinism:** the frozen spec clock and seeded `aql:rand` made
  property tests reproducible with explicit run/seed/shrink counts. Good.

---

## Recommendations, in priority order

1. **Fix the `comp/r` frame-cleanup over-pop (§1).** Higher-order code
   that re-invokes a captured function is a core idiom; it should not
   corrupt the caller's frame.
2. **Make the silent failures loud.** A bare-variable index into a List
   via `get` (§2) should raise, not return `None`; a wrong-shape
   `mat-mul` (§4) should raise a shape error. Silent wrong values are the
   single biggest time sink.
3. **Get `aql check` to zero false errors on a library in isolation
   (§5).** Treat the export map as a use site (kills false `unused_def`);
   accept gradual `Any` into typed dispatch without a hard `no_signature`;
   narrow `is`-guards symmetrically. Until then, document clearly that
   standalone `check` is advisory and the suite-level check is the gate.
4. **Reduce the `do`-vs-bare map-literal foot-gun (§3)** — either make
   `{k: [expr]}` evaluate consistently, or warn when a bracketed value is
   stored unevaluated.
5. **Ship a tagged release and a `go install`-able binary (§6).**

None of these are blockers — a complete, tested, three-surface-clean
library came out the other side — but each one is friction that the
language's own discipline elsewhere makes feel avoidable.

---

## Summary table

| # | Severity | Issue | Workaround today |
|---|----------|-------|------------------|
| 1 | 🔴 | `comp/r apply` frame cleanup over-pops on the 2nd call | avoid re-invoking a captured fn via a `comp/r` helper twice in one frame |
| 2 | 🔴 | `get`/`set` read a bare variable index as an atom (silent `None`) | parenthesise variable indices: `xs get (i)` |
| 3 | 🔴 | `{k: [expr]}` map value only evaluates under `do` | build such maps with `do {…}` |
| 4 | 🟡 | native multi-arg operand order surprising (`mat-mul` is B·A; `elem` is col,row) | verify against a known value; prefer `row`/`col` over `elem` |
| 5 | 🟡 | `aql check` reports false hard errors on a library in isolation | gate on `check` *through* the suites; standalone is advisory |
| 6 | 🟡 | no tagged release; `go install` blocked by replace directives | pin a commit; build from a codeload tarball |
| 7 | 🟢 | `print` chains reverse; 1-letter uppercase = type var; parse collapses `42.0`→`42`; List print comma vs space | one `print` per statement; lowercase value names; coerce on decode |

Overall: AQL is a pleasure to verify in and a hazard to mistype in. Close
the silent-failure gaps and make the checker gate-worthy, and the
day-to-day experience would match the quality of its execution-surface
guarantees.
