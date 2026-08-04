# Developer-experience report: stats on boru

**Date:** 2026-06-25
**boru build under test:** `boru-lang/boru` @ `12a44e0`
(`12a44e0c6ca3f49cd35a871b573fd96bc13d7fd6`, main as of 2026-06-24, PR
#189; built locally with `GOFLAGS=-mod=mod`; `boru -version` reports
`boru 12a44e0-main`).
**Context:** gotchas hit while building this statistics library — a new
module exercising `boru:matrix-util`, `boru:math-util`, the numeric types,
classes, and `do`/`error` against this build. All five suites
(`stats_*`) pass on it across the interpreter, `boru check` (0 errors),
and `boru --compile` (identical to the interpreter), enforced by
`test/divergence/run.sh`.

Severity: **🔴 high** (silent wrong results / crash / blocks a use case) ·
**🟡 medium** (friction, clear workaround) · **🟢 low** (papercut).

---

## Update (DX-driven boru fixes)

**2026-06-25.** This module was migrated to boru HEAD's new accessor
semantics and verified against a local boru build that carries three
upstream fixes (comp/r frame over-pop, StructUtil.parse float-fidelity,
and a checker `no_signature` fix). `get`/`getr` now *evaluate* their key,
so the literal bare-word field reads this library used — `(as-summary x)
get n`, `params get sigma`/`mu`, the `st`/`acc` map reads in `mode`, and
the linreg-field and error-`code` reads in the suites — moved to the new
dot sugar `recv.field` (with the quoted-atom `get field/q` form reserved
for the receiver-less, value-on-the-stack case); parenthesised computed
indexing such as `xs get (i)` is unchanged and intended. Because the
StructUtil.parse float-fidelity fix makes whole-valued Float moments
round-trip as Float (`8.0`, not `8`), the per-field `convert Float`
coercion that `Stats.decode` carried as the §5 workaround was removed and
the encode/decode round-trip stays green without it. All five `stats_*`
suites are green and `boru check` reports 0 errors on the module and on
each suite; this requires an boru build carrying these fixes — on older
builds the bare-word reads raise `undefined_word` and decode loses Float
type on whole-valued moments.

---

## 1. 🔴 `get`/`set` read a bare word index as an atom key, silently

`get`/`set` on a List or FlexList index, and on a Map/class field, share one
surface: the index/key argument is taken **literally**. A bare *word*
there is treated as an atom (a field name), not evaluated as a variable.
With a literal integer it works; with a variable it silently returns
`None`:

```boru
def s [10.0 20.0 30.0 40.0]
def i 1
s get 1     # => 20.0   (literal index: fine)
s get i     # => None   (variable read as the atom `i`, not its value 1)
s get (i)   # => 20.0   (parenthesise to force evaluation)
```

The damage is that `None` then flows into arithmetic and fails far from
the cause (`no matching signature for sub`, with a `None` receiver). The
fix is uniform: **parenthesise every variable index** — `xs get (i)`,
`arr set (j) (value)`. (Bare *field-name* keys are correct and intended:
`e get code`, `s set mean (m)`, `params get sigma`. The value position of
`set` evaluates a bare variable fine; only the index/key position is
literal.) `bloom.aql` already parenthesised its bit-array indices; this
report documents *why* for the order-statistic and solver code here.

---

## 2. 🔴 a map-literal value `{k: [expr]}` only evaluates under `do`

The bracketed-value form in a map literal is evaluated **only** when the
literal is prefixed with `do`. A bare `{…}` stores the brackets as a
literal List:

```boru
def v 23.0
def a {best: [v]}        #  a.best == [23.0]   (a one-element List!)
def b (do {best: [v]})   #  b.best == 23.0     (evaluated)
```

This bit the `mode` word: a seed state built with a bare `{best: [fs get
0] …}` carried `best` as `[23.0]` instead of `23.0`, so for all-distinct
data (where the fold never enters the `do {…}` update branch) `mode`
returned a List. A property test (`mode-is-a-member`) caught it. Fix:
build every map that uses the `[expr]` value form with `do {…}`. (The
bloom module only ever used `do {…}`, so it never saw this.)

---

## 3. 🟡 `MatrixUtil.mat-mul X Y` computes `Y @ X` (forward args reversed)

The two forward operands of `mat-mul` bind in reverse of the natural
reading order, so `mat-mul A B` is the product **B·A**, not A·B:

```boru
def amat (MatrixUtil.create [[1 2 3] [4 5 6]])   # 2x3
def bmat (MatrixUtil.create [[1 0] [0 1] [1 1]]) # 3x2
MatrixUtil.mat-mul amat bmat   # => Matrix(3x3) — i.e. bmat·amat, not amat·bmat
```

Easy to miss because the spec's own examples are square (2×2), where the
shape can't reveal the order. For `XᵀX` (covariance, OLS normal
equations) this means writing `MatrixUtil.mat-mul X (MatrixUtil.transpose
X)`. A wrong order here is a *silent* wrong-shape result, not an error,
so it only surfaces in a value check — both the covariance matrix and OLS
were initially wrong until pinned down against known answers.

## 4. 🟡 `MatrixUtil.elem` takes `(col, row)`, not `(row, col)`

`(m MatrixUtil.elem 1 0)` reads **column 1, row 0**. This *is* documented
in the module spec, but it inverts the usual `[row][col]` convention and
produced an out-of-bounds error before being spotted. This library reads
matrices a row at a time with `MatrixUtil.row` (naturally indexed) and
indexes into the resulting List, avoiding `elem` entirely.

## 5. 🟡 `StructUtil.parse` collapses whole-valued Floats to Integer

A snapshot field rendered as `42.0` parses back as Integer `42`, and a
sealed `class` field declared `Float` then rejects it
(`make: field "m2": expected Float … got Integer`). `Stats.decode`
coerces every numeric field (`(payload !. m2) convert Float`,
`… convert Integer` for `n`) so the round trip survives the
type collapse. (`bloom.aql` only round-tripped integer bit indices, so it
never hit this; its one Float, `p`, happened to be non-integral.)

---

## 6. 🟡 `boru check` mis-reports `Any`→typed dispatch as a hard error

A query word typed `[x:Any]` that dispatches `x` to a `List`-typed word
draws `no_signature: no matching signature …` — a hard **error**, not a
warning — even though it runs correctly (gradual `Any` should accept
anything). It surfaces inconsistently: the same call is clean when `x`
is provably a `List`, and errors when `x` is provably the *other* arm
(here a `Summary`). The shape that type-checks cleanly in both
directions: give the coercion helper a **union** param and narrow on the
**positive** branch —

```boru
def as-summary fn [
  [x:(List tor Summary)] [Summary] [
    if (x is List) [build-summary x] [x]   # `is List` (not `is Summary`) narrows cleanly
  ]
]
```

`if (x is Summary) [x] [build-summary x]` (negative-branch narrowing)
still errored; flipping to the positive `is List` test fixed it. This is
the same class of `boru check` false-positive the bloom report noted
(export-by-reference hides use sites); checked *through* a suite the
words type-check, which is what the gating `divergence` job asserts.

## 7. 🟢 the empty-list literal `[]` poisons downstream query types

At top level, `def acc (Stats.summary [] end)` then `Stats.mean acc end`
draws the §6 `no_signature` error, while `[1 2 3] Stats.summary` then
`mean` does not — the empty literal `[]` is typed loosely enough that the
checker can't carry it through the union param. It is invisible inside a
`Test.test`/`each` body (those are opaque to the checker), so only
top-level scripts see it. The empty constructor *runs* fine; the smoke
test simply seeds from non-empty data to keep `boru check` at 0 errors.

## 8. 🟢 single uppercase identifiers are parsed as type variables

`def M (MatrixUtil.create …)` fails with `type: body must be a type value
or literal, got Matrix(2x2)` — a one-letter uppercase name is taken as a
type variable. Bind matrices (and any value) to a lowercase name (`def
mat …`). Multi-letter PascalCase (`Summary`, used as a class) is fine.

## 9. 🟢 `print` forward-collection (carried over)

Unchanged from the bloom report: `print` collects a forward argument, so
chained `(a) print (b) print` reverses. Every print in this module uses
the one-value-per-statement idiom `print (value) end`.

---

## Observations

- **`MathUtil.floor`/`ceil` return `Integer` when applied to a Float**
  (`6.3 MathUtil.floor` ⇒ `6`, an Integer), so the result is directly
  usable as a list index — convenient for the quantile interpolation.
- **`boru:matrix-util` has no inverse/solve.** `mat-mul`, `transpose`,
  `det`, `dot`, `scale`, and the accessors are enough for covariance and
  correlation matrices, but OLS needs a linear solve, so this module
  ships a small Gaussian-elimination solver (partial pivoting) over
  `FlexList`s and feeds it `XᵀX` / `Xᵀy` built from the matrix words.
- **The DX feedback loop still shows.** The two 🔴 items here are
  call-convention sharp edges (literal index keys, `do`-gated map values)
  rather than interpreter bugs; both are catchable by property tests, and
  one was caught exactly that way.

---

## Summary

| # | Severity | Issue | Workaround |
|---|----------|-------|------------|
| 1 | 🔴 | `get`/`set` read a bare variable index as an atom key (silent `None`) | parenthesise variable indices: `xs get (i)` |
| 2 | 🔴 | `{k: [expr]}` map value only evaluates under `do` | build such maps with `do {…}` |
| 3 | 🟡 | `mat-mul X Y` is `Y·X` (silent wrong shape) | write `mat-mul X (transpose X)` for `XᵀX` |
| 4 | 🟡 | `MatrixUtil.elem` is `(col, row)` | use `MatrixUtil.row` + List indexing |
| 5 | 🟡 | `StructUtil.parse` collapses `42.0` → Integer | coerce field types on decode |
| 6 | 🟡 | `boru check` flags `Any`→typed dispatch as an error | union param + positive `is List` narrowing |
| 7 | 🟢 | empty `[]` literal poisons top-level query types in `boru check` | seed from non-empty data in checked scripts |
| 8 | 🟢 | one-letter uppercase names parse as type variables | bind values to lowercase names |
| 9 | 🟢 | `print` forward-collection reverses chains | one `print (value) end` per statement |

---

## Upgrade note — `main` @ 0721e8 (2026-07-11)

Re-evaluated against the then-newest `main` (`0721e8280e01`, 17 days past
the pinned `12a44e0`). **The pin stays at `12a44e0`** — `main` has landed
breaking changes that this module has not migrated to, and the fetch/CI
build paths were blocked in that session (details and the Go-module-proxy
workaround are in
[`aql-language-dx-report.md`](aql-language-dx-report.md#update--2026-07-11-re-evaluation-against-newer-main)).
What changes when the pin is eventually bumped:

- **#1 is fixed.** Bare `get` no longer treats a variable as an atom key —
  `xs get i` now evaluates `i`, so the silent-`None` trap is gone. The
  parenthesised `xs get (i)` this module uses keeps working.
- **New: `get` keys must be quoted.** The flip side: a *literal* key must
  be `get k/q` or `get "k"` — bare `get n` now raises `undefined word`.
  Every bare-atom `get` here (`get n`, `st get cur`, `e get code`, …)
  needs quoting, or a dot-access where the receiver is a class (`s.n`).
- **New: matrix types namespaced.** Bare `Matrix` is gone; the portable
  `fn`-param annotation is `Any` (dotted `MatrixUtil.Matrix` isn't valid
  in a param spec). The eight `[mat:Matrix]` / `[x:Matrix …]` signatures
  become `[mat:Any]`.
- **New: `Array` removed** (→ `FlexList`); the OLS solver's `make Array`
  becomes `make FlexList`, and its `convert List` becomes an `each`.
- **New: execution is check-gated and compiles by default.** `boru X` now
  refuses to run if the pre-flight check reports any error (`-no-check`
  escapes), and the default engine is the byte compiler (`-no-compile`
  escapes).

I applied all of the above and the descriptive/order/bivariate/
accumulator/distribution words then run correctly on `0721e8`. But the
**matrix/dataset words hit a wall**: with `[mat:Any]` the checker draws
false `no_signature` errors on them through an import, and the new
check-gate refuses to run them; even with `-no-check` the default
compiler mis-dispatches `iota` in that path. There is no valid matrix
annotation that satisfies the checker (bare `Matrix` gone, dotted names
rejected, aliases don't unify), so the block is **upstream**, not in this
module. The pin therefore stays at `12a44e0` — see
[`aql-language-dx-report.md`](aql-language-dx-report.md#update--2026-07-11-re-evaluation-against-newer-main)
for the full attempt. Revisit once `main` settles behind a tag.

All nine findings above still reproduce on `12a44e0`, where every suite
stays green across the interpreter, `boru check`, and the byte compiler.
