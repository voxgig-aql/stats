# AGENTS.md — using the `Stats` library

Guidance for an AI coding agent calling this statistics library from an
AQL project. Every code block below is verified to run against
`aql-lang/aql` @ `12a44e0` (main, which ships `aql:matrix-util`). If you
read nothing else, read [The one calling rule](#the-one-calling-rule) and
[Common mistakes](#common-mistakes).

## What it is

Descriptive, inferential, and matrix statistics. The public surface is
the `Stats` namespace plus the `Summary` type. There are two ways in:

- **Pure functions over a List** — `[1 2 3] Stats.mean end`. Simple,
  and the only way to get order statistics (median, quantile, mode).
- **A streaming `Summary` accumulator** — build one, `push` values into
  it, `merge` two of them, `encode`/`decode` it. It keeps running moments
  (Welford/Pébay), so mean/variance/skewness/kurtosis cost one pass and a
  merge is O(1). The same descriptive words read a `Summary` directly.

The **dataset** words take an `aql:matrix-util` `Matrix` whose rows are
observations and columns are variables.

## Import

```aql
import "./stats.aql"
```

- The path is resolved **relative to the working directory the script is
  run from**, not relative to the importing file.
- No `end` is needed after `import` on this build (a trailing `end` is
  harmless).
- Do **not** import `aql:math-util`, `aql:array-util`, `aql:matrix-util`,
  or `aql:struct-util` for the library's sake — `stats.aql` imports its
  own dependencies. **But** if *you* construct a `Matrix` to pass to the
  dataset words, add `import "aql:matrix-util"` to **your** script: the
  library's import does not re-export the `MatrixUtil` binding to callers.

## The one calling rule

AQL is not C/Python/JS. There is no `f(a, b)` and no `obj.method(a)`.
A call is written:

```
receiver Stats.verb arg1 arg2 end
```

— the **receiver/data comes first**, then the verb, then any extra
arguments, and the call is **terminated with `end`** (or wrapped in
parens). Without a terminator the verb can swallow whatever token
follows it and you get wrong results or a dispatch error.

```aql
def xs [2 4 4 4 5 5 7 9]
print ((xs Stats.mean   end)) end    # => 5.0
print ((xs Stats.median end)) end    # => 4.5
```

`(… )` parentheses count as a terminator, so `(xs Stats.mean)` is fine
too; use `end` for top-level statements that aren't already wrapped.

## API reference (exact call shapes)

Every **descriptive** word accepts either a `List` of numbers or a
`Summary`. The **order-statistic** words take a `List` only.

### Accumulator — the streaming `Summary`

| Call | Returns | Notes |
|------|---------|-------|
| `xs Stats.summary end` | `Summary` | Build from a List (`[]` ⇒ empty). |
| `s Stats.push x end` | the **same** `s` (mutated) | Add one observation. |
| `s Stats.push-all xs end` | the **same** `s` (mutated) | Add every element of a List. |
| `a Stats.merge b end` | the **same** `a` (mutated) | Combine `b`'s moments into `a`. Always compatible. |
| `s Stats.encode end` | `String` | jsonic snapshot of the moments. |
| `text Stats.decode end` | `Summary` | Rebuild from a snapshot; bad text raises `bad_payload`. |

### Descriptive (List **or** Summary)

| Call | Returns | Notes |
|------|---------|-------|
| `x Stats.count end` | `Integer` | Observation count; empty ⇒ `0`. |
| `x Stats.sum end` | `Float` | Total. |
| `x Stats.mean end` | `Float` | Needs ≥ 1 value. |
| `x Stats.variance end` | `Float` | **Sample** (n-1). Needs ≥ 2 values. |
| `x Stats.pvariance end` | `Float` | **Population** (n). |
| `x Stats.stddev end` / `x Stats.pstddev end` | `Float` | Sample / population std dev. |
| `x Stats.min end` / `x Stats.max end` / `x Stats.range end` | `Float` | Extremes and `max - min`. |
| `x Stats.skewness end` | `Float` | Biased g1. |
| `x Stats.kurtosis end` | `Float` | Biased **excess** g2. |

### Order statistics (List only)

| Call | Returns | Notes |
|------|---------|-------|
| `xs Stats.median end` | `Float` | |
| `xs Stats.quantile q end` | `Float` | `q` in `[0,1]`, linear interpolation (type-7). |
| `xs Stats.iqr end` | `Float` | Q3 − Q1. |
| `xs Stats.mode end` | `Float` | Most frequent; smallest value on a tie. |

### Bivariate (two Lists)

| Call | Returns | Notes |
|------|---------|-------|
| `xs Stats.covariance ys end` | `Float` | **Sample**. |
| `xs Stats.pcovariance ys end` | `Float` | **Population**. |
| `xs Stats.correlation ys end` | `Float` | Pearson r in `[-1, 1]`. |
| `xs Stats.linreg ys end` | `Map` | `{slope, intercept, r, r2}` — `xs` predictor, `ys` response. |

### Distributions / scores

| Call | Returns | Notes |
|------|---------|-------|
| `xs Stats.zscores end` | `List` | Sample-standardised values. |
| `x Stats.normal-pdf {mu, sigma} end` | `Float` | `sigma > 0`. |
| `x Stats.normal-cdf {mu, sigma} end` | `Float` | erf approximation (abs error ≈ 1.5e-7). |

### Matrix / dataset (rows = observations, cols = variables)

| Call | Returns | Notes |
|------|---------|-------|
| `mat Stats.col-means end` | `List` | Per-column means. |
| `mat Stats.col-variances end` / `mat Stats.col-stddevs end` | `List` | Per-column **sample** variance / std dev. |
| `mat Stats.cov-matrix end` | `Matrix` | **Sample** covariance matrix. Needs ≥ 2 rows. |
| `mat Stats.cor-matrix end` | `Matrix` | Correlation matrix. |
| `mat Stats.standardize end` | `Matrix` | Each column z-scored. |
| `x Stats.ols ys end` | `List` | Least-squares coefficients (`x` = design Matrix). |

Construct `Summary` values **only** through `Stats.summary`. Treat
`Summary` fields as read-only; mutate through the namespace words.

The unqualified `variance`/`stddev`/`covariance` are **sample**
statistics (Bessel's n-1 correction); the `p`-prefixed ones are
**population**.

Errors carry a code and message: catch with `do […] error […]` and read
`e get code` / `e get message` in the handler. Codes: `bad_input`
(empty/too-few data, a quantile out of `[0,1]`, a non-positive `sigma`,
mismatched lengths), `needs_data` (an order-statistic word called on a
`Summary`), `singular` (`Stats.ols` has no unique solution),
`bad_payload` (bad `Stats.decode` text).

## Copy-paste idioms (all verified)

Descriptive statistics over a List:

```aql
import "./stats.aql"
def xs [2 4 4 4 5 5 7 9]
print ((xs Stats.mean     end)) end   # => 5.0
print ((xs Stats.variance end)) end   # => 4.571428571428571 (sample)
print ((xs Stats.median   end)) end   # => 4.5
print ((xs Stats.quantile 0.9 end)) end   # => 7.6
```

The streaming accumulator — build, push, query:

```aql
def s ([1 2 3 4] Stats.summary end)
def _ (s Stats.push-all [5 6 7 8] end)
def _2 (s Stats.push 9 end)
print ((s Stats.count end)) end       # => 9
print ((s Stats.mean  end)) end       # => 5.0
```

Merge is O(1) and gives the same answer as pooling the data — useful for
parallel/streaming aggregation:

```aql
def a ([1 2 3 4] Stats.summary end)
def b ([5 6 7 8] Stats.summary end)
def merged (a Stats.merge b end)
print ((merged Stats.mean     end)) end   # => 4.5
print ((merged Stats.variance end)) end   # => 6.0
```

Persist and reload a Summary through the snapshot string:

```aql
def snap (merged Stats.encode end)
def back (snap Stats.decode end)
print ((back Stats.mean end)) end          # => 4.5
```

Bivariate and regression:

```aql
def x [1 2 3 4 5]
def y [2 4 5 4 5]
print ((x Stats.correlation y end)) end    # => 0.7745966692414834
def lr (x Stats.linreg y end)
print ((lr get slope)) end                 # => 0.6
print ((lr get intercept)) end             # => 2.2
```

Dataset statistics over a Matrix (import matrix-util yourself):

```aql
import "aql:matrix-util"
import "./stats.aql"
def mat (MatrixUtil.create [[1 2] [3 6] [5 10] [7 12]])
print ((mat Stats.col-means end)) end      # => [4.0 7.5]
def cov (mat Stats.cov-matrix end)         # => Matrix(2x2)
```

Multiple linear regression via OLS (prepend a 1s column for the
intercept):

```aql
import "aql:matrix-util"
import "./stats.aql"
def design (MatrixUtil.create [[1 1] [1 2] [1 3] [1 4]])
def coef (design Stats.ols [2 3 5 8] end)
print (coef) end                           # => [-0.5 2.0]  (intercept, slope)
```

Guard a misuse (an order statistic on a Summary raises `needs_data`):

```aql
def s ([1 2 3] Stats.summary end)
def code (do [s Stats.median end] error [ get code ])
print (code) end                           # => needs_data
```

## Common mistakes

| ✗ Don't write | ✓ Write | Why |
|---------------|---------|-----|
| `Stats.mean(xs)` | `xs Stats.mean end` | No `f(a,b)` syntax in AQL. |
| `xs.mean()` | `xs Stats.mean end` | No method-call syntax. |
| `xs Stats.mean` (no terminator, mid-expression) | `xs Stats.mean end` | The verb swallows the next token without `end`/parens. |
| `summary Stats.median end` | pass the raw **List** to `median` | Order statistics need the data; a `Summary` raises `needs_data`. |
| treat `Stats.variance` as population variance | `Stats.pvariance` for population | Bare `variance`/`stddev` are **sample** (n-1). |
| keep a pre-`push` copy of a Summary as "before" | `push`/`merge` mutate in place | The receiver and the returned value are the **same** object. |
| `xs get i` with a variable `i` | `xs get (i)` | A bare word after `get` is read as an atom key; parenthesise variable indices. |
| call the dataset words without `import "aql:matrix-util"` in your script | add the import yourself | The library's deps are not re-exported to callers. |
| `make Summary {…}` | `xs Stats.summary end` | Construct only via `Stats.summary`. |
| `"label" print (v) print` | `print (value) end`, one per statement | `print` collects a forward argument; chains print out of order. |

## Where to look next

- `docs/reference.md` — full signatures, semantics, complexity.
- `api.json` — the same API as a machine-readable manifest.
- `docs/how-to.md` — task recipes (summarise, merge, regress, persist).
- `docs/tutorial.md` — a guided first session.
- `test/stats_smoke_test.aql` — a complete, runnable worked example.
- `dx-report.md` — AQL-runtime gotchas observed while building this module.
