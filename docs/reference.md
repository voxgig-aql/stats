# Reference

Technical description of the `Stats` module's public surface. This page
is information-oriented: it states what each word is, its call shape,
arguments, and what it returns. For *why* the library behaves the way it
does, see [Explanation](explanation.md); for goal-directed recipes, see
the [How-to guides](how-to.md).

> **AI agents:** [AGENTS.md](../AGENTS.md) condenses the calling
> convention, idioms, and common mistakes for machine use; `api.json` is
> the same API as a machine-readable manifest.

The module exports a single namespace, `Stats`, plus the `Summary` type.
Import it with:

```aql
import "./stats.aql"
```

(No `end` is required after `import` on the pinned build; a trailing
`end` is harmless.) A consuming script does **not** need to import
`aql:math-util`, `aql:array-util`, `aql:matrix-util`, or
`aql:struct-util` itself — `stats.aql` imports them internally. The one
exception: scripts that build a `Matrix` to pass to the dataset words
must `import "aql:matrix-util"` themselves, because that binding is not
re-exported.

---

## Calling convention

Every operation is a forward-dispatched word: the **data/receiver comes
first**, then the verb, then any extra arguments, and the call is
**terminated with `end`** (or wrapped in parentheses), e.g.
`xs Stats.mean end` or `(xs Stats.quantile 0.9)`. Without a terminator
the word collects the following token as an argument. This is general
AQL forward-precedence behaviour, not specific to this module. There is
no `f(x)` or `x.f()` syntax.

Inputs may be Integer or Float; all arithmetic is done in Float, so
results are Float (`count` returns Integer). When indexing a List with a
variable, parenthesise the index: `xs get (i)`.

---

## Input polymorphism

The **descriptive** words (`count`, `sum`, `mean`, `variance`,
`pvariance`, `stddev`, `pstddev`, `min`, `max`, `range`, `skewness`,
`kurtosis`) accept **either a List of numbers or a `Summary`**. The
**order-statistic** words (`median`, `quantile`, `iqr`, `mode`) and the
**bivariate** words need the raw data and take a **List only** — calling
one on a Summary raises `needs_data`.

## Sample vs population

The unqualified `variance`/`stddev`/`covariance` are **sample**
statistics (divide by `n - 1`). The `p`-prefixed `pvariance`/`pstddev`/
`pcovariance` are **population** statistics (divide by `n`). The
per-column matrix words and `cov-matrix` are sample statistics.

---

## Types

### `Summary`

A sealed `class` instance — a streaming accumulator of running central
moments (Welford / Pébay). Fields:

| Field  | Type    | Meaning                                          |
|--------|---------|--------------------------------------------------|
| `n`    | Integer | Number of observations seen                      |
| `mean` | Float   | Running mean                                     |
| `m2`   | Float   | Central-moment sum Σ(x-mean)²                    |
| `m3`   | Float   | Central-moment sum Σ(x-mean)³                    |
| `m4`   | Float   | Central-moment sum Σ(x-mean)⁴                    |
| `min`  | Float   | Smallest observation                             |
| `max`  | Float   | Largest observation                              |

Instances are created **only** through `Stats.summary` (or rebuilt by
`Stats.decode`). Treat the fields as read-only; update exclusively
through the namespace words. Variance, skewness, and kurtosis are
derived from `m2`/`m3`/`m4` on demand. An empty Summary has `n = 0`;
`min`/`max` are seeded by the first observation.

---

## Accumulator words

### `Stats.summary`

Build a streaming accumulator from a List of numbers.

| | |
|--|--|
| **Call**    | `xs Stats.summary end` |
| **Args**    | `xs: List` |
| **Returns** | `Summary` (`[]` gives an empty Summary) |

### `Stats.push`

Add one observation. Welford update; **mutates** the Summary and returns
the same object.

| | |
|--|--|
| **Call**    | `s Stats.push x end` |
| **Args**    | `s: Summary`, `x: Number` |
| **Returns** | the same `Summary`, mutated |

### `Stats.push-all`

Add every element of a List.

| | |
|--|--|
| **Call**    | `s Stats.push-all xs end` |
| **Args**    | `s: Summary`, `xs: List` |
| **Returns** | the same `Summary`, mutated |

### `Stats.merge`

Combine the moments of `b` into `a` (parallel/Pébay combine).

| | |
|--|--|
| **Call**    | `a Stats.merge b end` |
| **Args**    | `a: Summary`, `b: Summary` |
| **Returns** | `a`, mutated to hold both |
| **Effect**  | `a` is mutated; `b` is unchanged. Always compatible (no parameters to disagree on). O(1). |

```aql
def a ([1 2 3 4] Stats.summary end)
def b ([5 6 7 8] Stats.summary end)
def _m (a Stats.merge b end)
print ((a Stats.mean end)) end   # => 4.5
```

### `Stats.encode`

Snapshot a Summary's moments as a jsonic String.

| | |
|--|--|
| **Call**    | `s Stats.encode end` |
| **Args**    | `s: Summary` |
| **Returns** | `String` (`{n, mean, m2, m3, m4, min, max}`) |

```aql
print (([1 2 3 4 5] Stats.summary end) Stats.encode end) end
# => {m2:10.0 m3:0.0 m4:34.0 max:5.0 mean:3.0 min:1.0 n:5}
```

Round-trips through `Stats.decode`.

### `Stats.decode`

Rebuild a Summary from an `encode` snapshot.

| | |
|--|--|
| **Call**    | `text Stats.decode end` |
| **Args**    | `text: String` |
| **Returns** | `Summary` |
| **Errors**  | `bad_payload` when the text is unparseable or missing a field |

Whole-valued Floats render without a decimal point, so `decode` coerces
each field's type back (Integer `n`, Float moments).

---

## Descriptive words (List | Summary)

| Word | Call | Returns | Semantics / errors |
|------|------|---------|--------------------|
| `count`     | `x Stats.count end`     | Integer | Number of observations; empty ⇒ `0`. |
| `sum`       | `x Stats.sum end`       | Float   | Total of the observations. |
| `mean`      | `x Stats.mean end`      | Float   | Arithmetic mean. Needs ≥ 1 value (else `bad_input`). |
| `variance`  | `x Stats.variance end`  | Float   | **Sample** variance, `m2/(n-1)`. Needs ≥ 2 values. |
| `pvariance` | `x Stats.pvariance end` | Float   | **Population** variance, `m2/n`. Needs ≥ 1 value. |
| `stddev`    | `x Stats.stddev end`    | Float   | **Sample** standard deviation (√variance). |
| `pstddev`   | `x Stats.pstddev end`   | Float   | **Population** standard deviation. |
| `min`       | `x Stats.min end`       | Float   | Minimum observation. |
| `max`       | `x Stats.max end`       | Float   | Maximum observation. |
| `range`     | `x Stats.range end`     | Float   | `max - min`. |
| `skewness`  | `x Stats.skewness end`  | Float   | Biased sample skewness g1 `= (m3/n)/(m2/n)^1.5`. `bad_input` on zero-variance data. |
| `kurtosis`  | `x Stats.kurtosis end`  | Float   | Biased excess kurtosis g2 `= (m4/n)/(m2/n)^2 - 3`. `bad_input` on zero-variance data. |

`x` is a `List` or a `Summary`.

```aql
def s ([2 4 4 4 5 5 7 9] Stats.summary end)
print ((s Stats.skewness end)) end   # => 0.6562500000000001
print ((s Stats.kurtosis end)) end   # => -0.21875
print ((s Stats.variance end)) end   # => 4.571428571428571
```

---

## Order statistics (List only)

A Summary discards the raw data, so these require a `List`; a Summary
raises `needs_data`.

| Word | Call | Returns | Semantics / errors |
|------|------|---------|--------------------|
| `median`   | `xs Stats.median end`     | Float | The 0.5 quantile. |
| `quantile` | `xs Stats.quantile q end` | Float | `q` in `[0,1]`, linear interpolation (NumPy/R type-7). `q` out of range ⇒ `bad_input`. |
| `iqr`      | `xs Stats.iqr end`        | Float | Inter-quartile range, Q3 − Q1. |
| `mode`     | `xs Stats.mode end`       | Float | Most frequent value; smallest such value on a tie. |

```aql
print (([1 2 3 4 5] Stats.quantile 0.25 end)) end   # => 2.0
```

---

## Bivariate words (two Lists)

Two equal-length Lists; mismatched lengths raise `bad_input`.

| Word | Call | Returns | Semantics / errors |
|------|------|---------|--------------------|
| `covariance`  | `xs Stats.covariance ys end`  | Float | **Sample** covariance. Needs ≥ 2 paired values. |
| `pcovariance` | `xs Stats.pcovariance ys end` | Float | **Population** covariance. |
| `correlation` | `xs Stats.correlation ys end` | Float | Pearson `r` in `[-1, 1]`. `bad_input` if a variable has zero variance. |
| `linreg`      | `xs Stats.linreg ys end`      | Map   | Simple regression of `ys` on `xs`: `{slope, intercept, r, r2}`. Predictor must have non-zero variance. |

```aql
def fit ([1 2 3 4 5] Stats.linreg [2 4 5 4 5] end)
print ((fit get slope)) end       # => 0.6
print ((fit get intercept)) end   # => 2.2
```

---

## Distribution / score words

| Word | Call | Returns | Semantics / errors |
|------|------|---------|--------------------|
| `zscores`    | `xs Stats.zscores end`               | List  | Sample-standardised `(x - mean)/stddev`. Zero stddev ⇒ `bad_input`. List only. |
| `normal-pdf` | `x Stats.normal-pdf {mu, sigma} end` | Float | Normal probability density at `x`. `sigma > 0` (else `bad_input`). |
| `normal-cdf` | `x Stats.normal-cdf {mu, sigma} end` | Float | Normal cumulative probability at `x`, via an Abramowitz–Stegun erf approximation (abs error ≈ 1.5e-7). `sigma > 0`. |

```aql
print ((0 Stats.normal-cdf {mu: 0.0, sigma: 1.0} end)) end   # => 0.5000000005
print (([1 2 3] Stats.zscores end)) end                      # => [-1.0, 0.0, 1.0]
```

---

## Matrix / dataset words

These take a `MatrixUtil` Matrix whose **rows are observations** and
**columns are variables**. The per-column words return a `List`;
`cov-matrix`/`cor-matrix`/`standardize` return a `Matrix`.

| Word | Call | Returns | Semantics / errors |
|------|------|---------|--------------------|
| `col-means`     | `mat Stats.col-means end`     | List   | Per-column means. |
| `col-variances` | `mat Stats.col-variances end` | List   | Per-column **sample** variances. |
| `col-stddevs`   | `mat Stats.col-stddevs end`   | List   | Per-column **sample** standard deviations. |
| `cov-matrix`    | `mat Stats.cov-matrix end`    | Matrix | **Sample** covariance matrix `(1/(n-1)) Xcᵀ Xc`. Needs ≥ 2 rows. |
| `cor-matrix`    | `mat Stats.cor-matrix end`    | Matrix | Correlation matrix derived from `cov-matrix`. |
| `standardize`   | `mat Stats.standardize end`   | Matrix | Each column z-scored (sample mean/stddev). |
| `ols`           | `x Stats.ols ys end`          | List   | Least-squares coefficients via `XᵀX b = Xᵀy`. Rows of `X` must match length of `ys` (else `bad_input`); a rank-deficient system raises `singular`. Prepend a 1s column for an intercept. |

```aql
import "aql:matrix-util"
import "./stats.aql"
def mat (MatrixUtil.create [[1 2] [3 6] [5 10] [7 12]])
print ((mat Stats.col-means end)) end                  # => [4.0, 7.5]
def design (MatrixUtil.create [[1 1] [1 2] [1 3] [1 4]])
print ((design Stats.ols [2 3 5 8] end)) end            # => [-0.5, 2.0]
```

---

## Errors at a glance

All failures raise coded errors; catch with `do […] error […]` and read
`e get code` / `e get message` (dispatch on several codes with `case`).

| Code | Situation |
|------|-----------|
| `bad_input` | empty data, too few points for the statistic, a `quantile` `q` outside `[0,1]`, a non-positive sigma, mismatched vector lengths, or a zero-variance predictor |
| `needs_data` | an order-statistic / bivariate word called on a Summary |
| `singular` | `Stats.ols` normal equations have no unique solution |
| `bad_payload` | `Stats.decode` text is not a `Stats.encode` snapshot |

A missing `end` after a `Stats.*` call is not a module error but a
general AQL dispatch problem — the word collects the following token
(add `end` or parens).
