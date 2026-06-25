---
name: stats-aql
description: Use when writing or editing AQL code that calls the Stats statistics library — Stats.mean / variance / stddev / median / quantile / mode / skewness / kurtosis / covariance / correlation / linreg / normal-pdf / normal-cdf / zscores, the streaming Stats.summary accumulator (push / merge / encode / decode), or the matrix/dataset words (col-means / cov-matrix / cor-matrix / standardize / ols), or any file that does `import "./stats.aql"`. Provides the exact AQL calling convention (which is not C/Python/JS), the API with its List-or-Summary and mutation semantics, verified copy-paste idioms, and fixes for the mistakes agents most often make (foreign call syntax like `Stats.mean(xs)`, missing `end` terminators, calling order statistics on a Summary).
---

# Calling the Stats statistics library (AQL)

Descriptive, inferential, and matrix statistics. Public surface = the
`Stats` namespace plus the `Summary` type. Everything below is verified
against `aql @ 12a44e0` (main, with `aql:matrix-util`).

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

AQL has no `f(a, b)` and no `obj.method(a)`. Write:

```
receiver Stats.verb arg1 arg2 end
```

Receiver/data first, then the verb, then any extra args, **terminated
with `end`** (or wrap the whole call in parens). Without a terminator the
verb swallows the following token — wrong result or a dispatch error.

```aql
print (([1 2 3 4 5] Stats.mean end)) end          # => 3.0
print (([2 4 4 4 5 5 7 9] Stats.median end)) end   # => 4.5
```

## API (exact call shapes)

Every **descriptive** word takes a `List` of numbers **or** a `Summary`.
The **order-statistic** words take a `List` only.

| Call | Returns | Notes |
|------|---------|-------|
| `xs Stats.summary end` | `Summary` | Build a streaming accumulator from a List (`[]` ⇒ empty). |
| `s Stats.push x end` / `s Stats.push-all xs end` | `Summary` | **Mutates** `s` in place, returns it. |
| `a Stats.merge b end` | `Summary` | Combine moments of `b` into `a` (mutates `a`). Always compatible. |
| `s Stats.encode end` / `text Stats.decode end` | `String` / `Summary` | jsonic snapshot round-trip; bad text raises `bad_payload`. |
| `x Stats.mean / sum / count / min / max / range end` | `Float`/`Integer` | `x` = List or Summary. |
| `x Stats.variance / stddev end` | `Float` | **Sample** (n-1). Needs ≥ 2 values. |
| `x Stats.pvariance / pstddev end` | `Float` | **Population** (n). |
| `x Stats.skewness / kurtosis end` | `Float` | Biased g1 / excess g2. |
| `xs Stats.median end` | `Float` | List only. |
| `xs Stats.quantile q end` | `Float` | `q` in `[0,1]`, linear interpolation. List only. |
| `xs Stats.iqr / mode end` | `Float` | List only. |
| `xs Stats.covariance ys end` / `pcovariance` | `Float` | Sample / population. |
| `xs Stats.correlation ys end` | `Float` | Pearson r. |
| `xs Stats.linreg ys end` | `Map` | `{slope, intercept, r, r2}` (xs predictor, ys response). |
| `xs Stats.zscores end` | `List` | Sample-standardised. |
| `x Stats.normal-pdf {mu, sigma} end` / `normal-cdf` | `Float` | `sigma > 0`. CDF via an erf approximation (~1e-7). |
| `mat Stats.col-means / col-variances / col-stddevs end` | `List` | Per-column (rows = observations). |
| `mat Stats.cov-matrix / cor-matrix / standardize end` | `Matrix` | Sample covariance / correlation / z-scored columns. |
| `x Stats.ols ys end` | `List` | Least-squares coefficients (x = design Matrix; prepend a 1s column for an intercept). |

Catch errors with `do […] error […]`; read `e get code`. Codes:
`bad_input` (empty/too-few data, q out of range, sigma ≤ 0, length
mismatch), `needs_data` (order statistic called on a Summary),
`singular` (OLS has no unique solution), `bad_payload` (bad decode).

## Idioms (verified)

```aql
import "./stats.aql"
def xs [2 4 4 4 5 5 7 9]
print ((xs Stats.mean end))     end   # => 5.0
print ((xs Stats.variance end)) end   # => 4.571428571428571 (sample)
print ((xs Stats.median end))   end   # => 4.5
```

Streaming accumulator — build once, query many; merge is O(1):

```aql
def a ([1 2 3 4] Stats.summary end)
def b ([5 6 7 8] Stats.summary end)
def m (a Stats.merge b end)
print ((m Stats.mean end)) end        # => 4.5
def back ((m Stats.encode end) Stats.decode end)   # persist + reload
```

Dataset (rows = observations, cols = variables):

```aql
import "aql:matrix-util"
def mat (MatrixUtil.create [[1 2] [3 6] [5 10] [7 12]])
print ((mat Stats.col-means end)) end          # => [4.0 7.5]
def cov (mat Stats.cov-matrix end)             # => Matrix(2x2)
```

## Common mistakes

| ✗ Don't | ✓ Do | Why |
|---------|------|-----|
| `Stats.mean(xs)` / `xs.mean()` | `xs Stats.mean end` | AQL has no call/method syntax. |
| `xs Stats.mean` mid-expression, no terminator | `xs Stats.mean end` | The verb swallows the next token. |
| `summary Stats.median end` | pass the raw **List** | Order statistics need the data; a Summary raises `needs_data`. |
| treat `Stats.variance` as population | `Stats.pvariance` for population | Bare `variance`/`stddev` are **sample** (n-1). |
| keep a pre-`push` copy of a Summary | none — `push`/`merge` mutate in place | The argument and the return value are the same object. |
| `xs get i` with a variable `i` | `xs get (i)` | Bare words after `get` are read as atom keys; parenthesise variable indices. |
| build a Matrix without importing matrix-util | `import "aql:matrix-util"` in your script | The library's deps are not re-exported to callers. |
| `"label" print (v) print` | `print (v) end`, one per statement | `print` collects forward; chains print out of order. |

If the full repo is available, `AGENTS.md`, `api.json` (machine-readable
signatures), and `docs/reference.md` have the complete guide;
`test/stats_smoke_test.aql` is a runnable example.
