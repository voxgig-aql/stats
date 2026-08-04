# How-to guides

Task-oriented recipes. Each one assumes you already know roughly what
the statistic is; if you want a guided first session, start with the
[Tutorial](tutorial.md). For the *why* behind any of these, follow the
links into the [Explanation](explanation.md); for exact signatures, the
[Reference](reference.md).

- [Install and run boru](#install-and-run-aql)
- [Summarise a List of numbers](#summarise-a-list-of-numbers)
- [Choose sample vs population](#choose-sample-vs-population)
- [Order statistics: median, quantile, IQR, mode](#order-statistics-median-quantile-iqr-mode)
- [Stream data with a Summary](#stream-data-with-a-summary)
- [Persist a Summary with encode/decode](#persist-a-summary-with-encodedecode)
- [Correlation and linear regression](#correlation-and-linear-regression)
- [Dataset statistics over a Matrix](#dataset-statistics-over-a-matrix)
- [Multiple regression with OLS](#multiple-regression-with-ols)
- [Handle errors](#handle-errors)
- [Use the library from your own script](#use-the-library-from-your-own-script)
- [Run the tests](#run-the-tests)

---

## Install and run boru

The module is written in boru, which has no tagged release yet, so build
the interpreter from source (the documented `go install …/aql@latest`
fails on the repo's replace directives). The pinned commit is
`618562025d9e0154107306927911a8b1b046333c` (main):

```bash
curl -fsSL https://codeload.github.com/boru-lang/boru/tar.gz/618562025d9e0154107306927911a8b1b046333c | tar -xz
cd aql-618562025d9e0154107306927911a8b1b046333c/cmd/go
GOFLAGS=-mod=mod go build -o "$HOME/.local/bin/boru" ./boru
```

Or clone and check the ref out, then build the same `cmd/go` target:

```bash
git clone https://github.com/boru-lang/boru
git -C boru checkout 618562025d9e0154107306927911a8b1b046333c
(cd aql/cmd/go && GOFLAGS=-mod=mod go build -o "$HOME/.local/bin/boru" ./boru)
```

Make sure `$HOME/.local/bin` is on your `PATH`, then check it:

```bash
boru -version    # => boru 6185620-main
```

Run any script in this repo by passing its path:

```bash
boru test/stats_smoke_test.aql
```

This module is verified against boru commit `6185620`; `boru:matrix-util`
and the runtime behaviours this library relies on require this ref or
newer. In remote Claude Code sessions a SessionStart hook builds it
automatically.

---

## Summarise a List of numbers

Every descriptive word is written verb first, then the data, then `end`.
Inputs may be Integer or Float; results are Float (`count` is Integer):

```boru
import "./stats.aql"
def xs [2 4 4 4 5 5 7 9]
print ((Stats.mean xs end)) end     # => 5.0
print ((Stats.stddev xs end)) end   # => 2.138089935299395
print ((Stats.range xs end)) end    # => 7.0
```

The full descriptive set is `count`, `sum`, `mean`, `variance`,
`pvariance`, `stddev`, `pstddev`, `min`, `max`, `range`, `skewness`,
`kurtosis` — all listed in the
[Reference](reference.md#descriptive-words-list--summary).

---

## Choose sample vs population

The unqualified `variance`/`stddev` divide by `n - 1` (the **sample**
estimators, with Bessel's correction); the `p`-prefixed `pvariance`/
`pstddev` divide by `n` (the **population** values). Use the sample form
when your data is a sample from a larger population (the usual case);
use the population form when your data *is* the entire population.

```boru
import "./stats.aql"
def xs [2 4 4 4 5 5 7 9]
print (`sample var:     ${(Stats.variance xs end)}`) end   # => 4.571428571428571
print (`population var: ${(Stats.pvariance xs end)}`) end  # => 3.9999999999999996
print (`sample sd:      ${(Stats.stddev xs end)}`) end     # => 2.138089935299395
print (`population sd:  ${(Stats.pstddev xs end)}`) end    # => 1.9999999999999998
```

The same split applies to `covariance` (sample) vs `pcovariance`
(population). Why `n - 1`: [Explanation → Sample vs
population](explanation.md#sample-vs-population).

---

## Order statistics: median, quantile, IQR, mode

These read sorted positions out of the data, so they need the **raw
List** — they cannot run off a Summary (that raises `needs_data`):

```boru
import "./stats.aql"
def xs [2 4 4 4 5 5 7 9]
print (`median:   ${(Stats.median xs end)}`) end          # => 4.5
print (`q90:      ${(Stats.quantile xs 0.9 end)}`) end     # => 7.6
print (`iqr:      ${(Stats.iqr xs end)}`) end              # => 1.5
print (`mode:     ${(Stats.mode xs end)}`) end             # => 4.0
```

`quantile` takes `q` in `[0, 1]` and interpolates linearly
(NumPy/R type-7); a `q` outside that range raises `bad_input`. `iqr` is
`quantile 0.75 - quantile 0.25`. `mode` returns the most frequent value
(the smallest such value on a tie).

---

## Stream data with a Summary

When data arrives incrementally, or you want a single mergeable object,
build a `Summary` and push into it. The accumulator is mutated **in
place** and returned, so bind the result to a throwaway name:

```boru
import "./stats.aql"
def s (Stats.summary [] end)          # [] gives an empty Summary
def _1 (Stats.push 10 s end)          # one value (accumulator s last)
def _2 (Stats.push-all [20 30 40] s end)  # many at once
print (`mean: ${(Stats.mean s end)} n: ${(Stats.count s end)}`) end
# => mean: 25.0 n: 4
```

Two Summaries combine in O(1) — and exactly, not approximately:

```boru
def a (Stats.summary [1 2 3 4] end)
def b (Stats.summary [5 6 7 8] end)
def _m (Stats.merge b a end)
print (`merged mean: ${(Stats.mean a end)}`) end   # => 4.5
```

`merge` folds `b` into `a` and returns `a` (the receiver `a` is the last
argument); `b` is left untouched. All
the descriptive words accept a Summary; the order-statistic and
bivariate words do not (they need the raw data). Background:
[Explanation → Why a streaming Summary](explanation.md#why-a-streaming-summary).

---

## Persist a Summary with encode/decode

`Stats.encode` snapshots a Summary's moments as a jsonic string;
`Stats.decode` rebuilds an equivalent Summary:

```boru
import "./stats.aql"
def s (Stats.summary [1 2 3 4 5] end)
def snap (Stats.encode s end)
print (snap) end
# => {m2:10.0 m3:0.0 m4:34.0 max:5.0 mean:3.0 min:1.0 n:5}
def back (Stats.decode snap end)
print ((Stats.mean back end)) end   # => 3.0
```

The snapshot carries the count, mean, central-moment sums (`m2`, `m3`,
`m4`), and the running `min`/`max` — everything the descriptive words
need, but **not** the raw data, so a decoded Summary still can't answer
the order-statistic words. Malformed text (not parseable, or missing a
field) raises `bad_payload`.

---

## Correlation and linear regression

Bivariate words take two equal-length Lists:

```boru
import "./stats.aql"
def xs [1 2 3 4 5]
def ys [2 4 5 4 5]
print (`covariance:  ${(Stats.covariance xs ys end)}`) end    # => 1.5
print (`correlation: ${(Stats.correlation xs ys end)}`) end   # => 0.7745966692414833
def fit (Stats.linreg xs ys end)
print (`slope:     ${(fit get slope)}`) end       # => 0.6
print (`intercept: ${(fit get intercept)}`) end    # => 2.2
print (`r2:        ${(fit get r2)}`) end            # => 0.6000000000000001
```

`linreg` returns a Map with `slope`, `intercept`, `r` (Pearson), and
`r2`. Mismatched lengths raise `bad_input`; a predictor with zero
variance raises `bad_input` (correlation/regression are undefined).

---

## Dataset statistics over a Matrix

The dataset words operate on a `MatrixUtil` Matrix whose **rows are
observations** and **columns are variables**. Scripts that build a
Matrix must `import "boru:matrix-util"` themselves — the library does not
re-export the `MatrixUtil` binding:

```boru
import "boru:matrix-util"
import "./stats.aql"
def mat (MatrixUtil.create [[1 2] [3 6] [5 10] [7 12]])
print (`col-means:    ${(Stats.col-means mat end)}`) end
# => [4.0 7.5]
print (`col-stddevs:  ${(Stats.col-stddevs mat end)}`) end
# => [2.581988897471611 4.43471156521669]
def cov (Stats.cov-matrix mat end)
print (`cov row 0:    ${(cov MatrixUtil.row 0)}`) end
# => [6.666666666666666 11.333333333333332]
def cor (Stats.cor-matrix mat end)
print (`cor row 0:    ${(cor MatrixUtil.row 0)}`) end
# => [1.0 0.9897782665572892]
```

`col-means`/`col-variances`/`col-stddevs` return a List (one entry per
column); `cov-matrix`/`cor-matrix`/`standardize` return a Matrix. The
per-column variance and the covariance matrix are **sample** statistics
(`1/(n-1)`). `cov-matrix` needs at least 2 rows. How these map to matrix
algebra: [Explanation → Dataset words as matrix
algebra](explanation.md#dataset-words-as-matrix-algebra).

---

## Multiple regression with OLS

`Stats.ols` fits least-squares coefficients for a design Matrix `X`
(rows = observations, cols = predictors) and a response List `y`, via
the normal equations. **Prepend a column of 1s** to `X` for an
intercept term:

```boru
import "boru:matrix-util"
import "./stats.aql"
# columns: [intercept x1 x2]; y was generated from  y = 1 + 2*x1 + 0.5*x2
def x (MatrixUtil.create [[1 1 2] [1 2 1] [1 3 4] [1 4 3] [1 5 6]])
def y [4.0 5.5 9.0 10.5 14.0]
print (`coeffs: ${(Stats.ols x y end)}`) end
# => [1.0 2.0 0.5000000000000001]   (intercept, x1, x2)
```

The result is a List of coefficients in column order. The rows of `X`
must match the length of `y` (else `bad_input`); a rank-deficient system
— e.g. a duplicated or constant predictor column — raises `singular`.
How OLS solves the normal equations:
[Explanation → How OLS works](explanation.md#how-ols-works).

---

## Handle errors

Failures raise coded errors. Catch them with `do [...] error [...]`;
inside the handler the Error value is on the stack, so read `get code` /
`get message` (dispatch on the code with `case` if you handle several):

```boru
import "./stats.aql"
def msg (do [Stats.variance [5] end] error [get message])
print (msg) end
# => Stats: need at least 2 value(s) (have 1)
def code (do [Stats.variance [5] end] error [get code])
print (code) end                 # => bad_input

# an order statistic on a Summary needs the raw data:
def s (Stats.summary [1 2 3] end)
print ((do [Stats.median s end] error [get code])) end   # => needs_data

# unparseable decode payload:
print ((do [Stats.decode "not a snapshot" end] error [get code])) end
# => bad_payload
```

The four codes:

| Code | When |
|------|------|
| `bad_input` | empty/too-few data, `quantile` `q` outside `[0,1]`, non-positive sigma, mismatched vector lengths, zero-variance predictor |
| `needs_data` | an order-statistic / bivariate word called on a Summary |
| `singular` | `Stats.ols` normal equations have no unique solution |
| `bad_payload` | `Stats.decode` text is not a `Stats.encode` snapshot |

In a test, assert the failure (or its exact code) with `boru:test`:

```boru
import "boru:test"
import "./stats.aql"
[Stats.variance [5] end] Assert.throws end
def e (do [Stats.variance [5] end])
bad_input/q (e get code) Assert.equal end
```

(Why the module raises coded errors:
[Explanation → Raising errors](explanation.md#raising-errors).)

---

## Use the library from your own script

Import the library by relative path; you do **not** need to import
`boru:math-util`, `boru:array-util`, `boru:matrix-util`, or
`boru:struct-util` yourself — `stats.aql` pulls in its own dependencies:

```boru
import "./stats.aql"

def xs [1 2 3 4 5]
# … use the Stats namespace …
```

The one exception is when *your* script builds a `Matrix` to hand to the
dataset words (`col-means`, `cov-matrix`, `ols`, …): add
`import "boru:matrix-util"` yourself, because the `MatrixUtil` binding is
not re-exported. (No `end` is needed after `import` on the pinned build;
the import path resolves relative to the directory you run the script
from.) Every `Stats.*` call must end with `end` (or be wrapped in
parens) so the word doesn't swallow the following token.
`test/stats_smoke_test.aql` is a complete worked example you can copy
from.

---

## Run the tests

Five suites ship with the module. Run them with `boru`:

```bash
boru test/stats_unit_test.aql   # example-based unit tests — direct (boru:test)
boru test/stats_unit_spec.aql   # example-based unit tests — declarative spec format
boru test/stats_prop_test.aql   # property tests — direct Test.check-prop form
boru test/stats_prop_spec.aql   # property tests — declarative spec format
boru test/stats_smoke_test.aql  # end-to-end walk-through over every public word
```

The file names follow a consistent convention: `_test.aql` is a direct
suite (assertions or `Test.check-prop` calls written out in code), and
`_spec.aql` is a declarative suite (cases or properties built as data
and handed to a runner). Both the unit and property layers ship in both
forms.

The two unit suites express the same example checks two ways:
`stats_unit_test.aql` asserts imperatively with `Test.test` /
`Assert.equal`, while `stats_unit_spec.aql` builds each check as a
`TestSpec` (`Test.spec` / `Test.case`) that `Test.run-spec` dispatches.

The two property suites are likewise split: `stats_prop_spec.aql` builds
each property as a declarative `PropertySpec` (`Test.prop`) and runs it
with `Test.run-property` at the default 100 iterations — clean, but the
run count is fixed. `stats_prop_test.aql` calls the imperative
`Test.check-prop` driver directly, passing `runs`/`seed`/`max-shrinks`
explicitly, which is why it carries the costlier properties (merge
equivalence, encode/decode round-trips) at a tuned run budget.

Each assertion-bearing suite ends by asserting `Test.fail-count` is `0`
and prints `all green`, so a failure makes `boru` exit non-zero — which
is exactly what the [CI workflow](../.github/workflows/test.yml) checks on every push
and pull request.

One more check sits outside this set. `test/divergence/` runs every
suite through all three of boru's execution surfaces — the interpreter,
`boru check` (static type-check), and the byte compiler (`boru --compile`)
— and asserts none errors or disagrees. Run it with:

```bash
test/divergence/run.sh
```

It builds a newer boru (the `--compile` CLI postdates this module's pin)
and prints a per-suite interpreter/check/bytecode matrix. See
[`test/divergence/README.md`](../test/divergence/README.md) for the one
upstream byte-compiler bug this guards against (a compiled `each` body
drops a *block-local* binding) and the structural choice that keeps the
suites clear of it.
