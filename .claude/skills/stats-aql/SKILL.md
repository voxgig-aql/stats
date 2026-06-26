---
name: stats-aql
description: Use when writing or editing AQL code that calls the Stats statistics library — Stats.mean / variance / stddev / median / quantile / mode / skewness / kurtosis / covariance / correlation / linreg / normal-pdf / normal-cdf / zscores, the streaming Stats.summary accumulator (push / merge / encode / decode), or the matrix/dataset words (col-means / cov-matrix / cor-matrix / standardize / ols), or any file that does `import "./stats.aql"`. Provides the exact AQL calling convention (which is not C/Python/JS), the API with its List-or-Summary and mutation semantics, verified copy-paste idioms, and fixes for the mistakes agents most often make (foreign call syntax like `Stats.mean(xs)`, missing `end` terminators, calling order statistics on a Summary).
---

# Calling the Stats statistics library (AQL)

Descriptive, inferential, and matrix statistics. Public surface = the
`Stats` namespace plus the `Summary` type. Everything below is verified
against `aql @ 2342477` (main, with `aql:matrix-util`).

## Import

```aql
import "./stats.aql"
```

- Path resolves relative to the **working directory the script runs
  from**, not the importing file.
- No `end` is needed after `import`.
- The library imports its own deps (`aql:math-util`, `aql:array-util`,
  `aql:matrix-util`, `aql:struct-util`). But if **you** build a `Matrix`
  to pass to the dataset words, add `import "aql:matrix-util"` yourself —
  the library's import does not re-export the `MatrixUtil` binding.

## The one calling rule

AQL has no `f(a, b)` and no `obj.method(a)`. A `Stats` call is written
**verb-first** (the all-forward form):

```
Stats.verb data arg1 arg2 end
```

Verb first, then the data, then any extra args, **terminated with `end`**
(or wrap the whole call in parens). Without a terminator the verb
swallows the following token — wrong result or a dispatch error.

```aql
print ((Stats.mean [1 2 3 4 5] end)) end          # => 3.0
print ((Stats.median [2 4 4 4 5 5 7 9] end)) end   # => 4.5
```

(The core words `each`/`fold`, indexing `get`, and the matrix accessors
`MatrixUtil.row`/`col` read their subject from the stack and stay
subject-first: `xs get (i)`, `mat MatrixUtil.col j`.)

## API (exact call shapes)

Every **descriptive** word takes a `List` of numbers **or** a `Summary`.
The **order-statistic** words take a `List` only.

| Call | Returns | Notes |
|------|---------|-------|
| `Stats.summary xs end` | `Summary` | Build a streaming accumulator from a List (`[]` ⇒ empty). |
| `Stats.push s x end` / `Stats.push-all s xs end` | `Summary` | **Mutates** `s` in place, returns it. |
| `Stats.merge a b end` | `Summary` | Combine moments of `b` into `a` (mutates `a`). Always compatible. |
| `Stats.encode s end` / `Stats.decode text end` | `String` / `Summary` | jsonic snapshot round-trip; bad text raises `bad_payload`. |
| `Stats.mean x end` / `sum` / `count` / `min` / `max` / `range` | `Float`/`Integer` | `x` = List or Summary. |
| `Stats.variance x end` / `stddev x end` | `Float` | **Sample** (n-1). Needs ≥ 2 values. |
| `Stats.pvariance x end` / `pstddev x end` | `Float` | **Population** (n). |
| `Stats.skewness x end` / `kurtosis x end` | `Float` | Biased g1 / excess g2. |
| `Stats.median xs end` | `Float` | List only. |
| `Stats.quantile xs q end` | `Float` | `q` in `[0,1]`, linear interpolation. List only. |
| `Stats.iqr xs end` / `mode xs end` | `Float` | List only. |
| `Stats.covariance xs ys end` / `pcovariance xs ys end` | `Float` | Sample / population. |
| `Stats.correlation xs ys end` | `Float` | Pearson r. |
| `Stats.linreg xs ys end` | `Map` | `{slope, intercept, r, r2}` (xs predictor, ys response). |
| `Stats.zscores xs end` | `List` | Sample-standardised. |
| `Stats.normal-pdf x {mu, sigma} end` / `normal-cdf x {mu, sigma} end` | `Float` | `sigma > 0`. CDF via an erf approximation (~1e-7). |
| `Stats.col-means mat end` / `col-variances` / `col-stddevs` | `List` | Per-column (rows = observations). |
| `Stats.cov-matrix mat end` / `cor-matrix` / `standardize` | `Matrix` | Sample covariance / correlation / z-scored columns. |
| `Stats.ols x ys end` | `List` | Least-squares coefficients (x = design Matrix; prepend a 1s column for an intercept). |

Catch errors with `do […] error […]`; read `e get code`. Codes:
`bad_input` (empty/too-few data, q out of range, sigma ≤ 0, length
mismatch), `needs_data` (order statistic called on a Summary),
`singular` (OLS has no unique solution), `bad_payload` (bad decode).

## Idioms (verified)

```aql
import "./stats.aql"
def xs [2 4 4 4 5 5 7 9]
print ((Stats.mean xs end))     end   # => 5.0
print ((Stats.variance xs end)) end   # => 4.571428571428571 (sample)
print ((Stats.median xs end))   end   # => 4.5
```

Streaming accumulator — build once, query many; merge is O(1):

```aql
def a (Stats.summary [1 2 3 4] end)
def b (Stats.summary [5 6 7 8] end)
def m (Stats.merge a b end)
print ((Stats.mean m end)) end        # => 4.5
def back (Stats.decode (Stats.encode m end) end)   # persist + reload
```

Dataset (rows = observations, cols = variables):

```aql
import "aql:matrix-util"
def mat (MatrixUtil.create [[1 2] [3 6] [5 10] [7 12]])
print ((Stats.col-means mat end)) end          # => [4.0 7.5]
def cov (Stats.cov-matrix mat end)             # => Matrix(2x2)
```

## Common mistakes

| ✗ Don't | ✓ Do | Why |
|---------|------|-----|
| `Stats.mean(xs)` / `xs.mean()` | `Stats.mean xs end` | AQL has no call/method syntax. |
| `Stats.mean xs` mid-expression, no terminator | `Stats.mean xs end` | The verb swallows the next token. |
| `Stats.median s end` on a Summary | pass the raw **List** | Order statistics need the data; a Summary raises `needs_data`. |
| treat `Stats.variance` as population | `Stats.pvariance` for population | Bare `variance`/`stddev` are **sample** (n-1). |
| keep a pre-`push` copy of a Summary | none — `push`/`merge` mutate in place | The argument and the return value are the same object. |
| `xs get i` with a variable `i` | `xs get (i)` | Bare words after `get` are read as atom keys; parenthesise variable indices. |
| build a Matrix without importing matrix-util | `import "aql:matrix-util"` in your script | The library's deps are not re-exported to callers. |
| `"label" print (v) print` | `print (v) end`, one per statement | `print` collects forward; chains print out of order. |

If the full repo is available, `AGENTS.md`, `api.json` (machine-readable
signatures), and `docs/reference.md` have the complete guide;
`test/stats_smoke_test.aql` is a runnable example.
