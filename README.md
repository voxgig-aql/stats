# stats

A **statistics library** implemented in
[AQL](https://github.com/aql-lang/aql) — descriptive, inferential, and
matrix statistics behind a single `Stats` namespace, plus a streaming
`Summary` accumulator. It builds on AQL's numeric types and the
`aql:matrix-util` / `aql:math-util` modules.

```aql
import "./stats.aql"

def xs [2 4 4 4 5 5 7 9]
print ((xs Stats.mean   end)) end   # => 5.0
print ((xs Stats.median end)) end   # => 4.5
print ((xs Stats.stddev end)) end   # => 2.138089935299395  (sample)
```

> **Calling this library from an AI coding agent?** Read
> **[AGENTS.md](AGENTS.md)** first — the exact AQL calling convention,
> verified idioms, and common mistakes. (Claude Code auto-loads it via
> `CLAUDE.md`; a portable skill lives in
> [`.claude/skills/stats-aql`](.claude/skills/stats-aql/SKILL.md).)

## Two ways in

**Pure functions over a List** — the simplest path, and the only way to
get order statistics:

```aql
print (([1 2 3 4 5] Stats.variance end)) end       # => 2.5 (sample)
print (([1 2 3 4 5] Stats.quantile 0.9 end)) end   # => 4.6
```

**A streaming `Summary` accumulator** — keeps running Welford moments, so
mean/variance/skewness/kurtosis cost one pass and two summaries merge in
O(1) (ideal for parallel or streaming aggregation):

```aql
def a ([1 2 3 4] Stats.summary end)
def b ([5 6 7 8] Stats.summary end)
def merged (a Stats.merge b end)
print ((merged Stats.mean end)) end                # => 4.5
```

The dataset words operate on an `aql:matrix-util` Matrix (rows =
observations, columns = variables) — column means, covariance and
correlation matrices, standardization, and ordinary least squares.

## Documentation

The docs follow the [Diátaxis](https://diataxis.fr) framework — four
modes, each serving a different need:

| | Mode | Read this when you want to… |
|--|------|----------------------------|
| 🎓 | **[Tutorial](docs/tutorial.md)** | learn by working a first session step by step |
| 🔧 | **[How-to guides](docs/how-to.md)** | accomplish a specific task (summarise, merge, regress, persist…) |
| 📖 | **[Reference](docs/reference.md)** | look up exact words, signatures, and return types |
| 💡 | **[Explanation](docs/explanation.md)** | understand how it works and why it's built this way |

New here? Read the [Tutorial](docs/tutorial.md). Already know the domain
and just want the API? Jump to the [Reference](docs/reference.md).

## The `Stats` API at a glance

| Group | Words |
|-------|-------|
| Accumulator | `xs Stats.summary` · `s Stats.push x` · `s Stats.push-all xs` · `a Stats.merge b` · `s Stats.encode` · `text Stats.decode` |
| Descriptive (List or Summary) | `mean` · `sum` · `count` · `min` · `max` · `range` · `variance`/`pvariance` · `stddev`/`pstddev` · `skewness` · `kurtosis` |
| Order statistics (List) | `median` · `quantile q` · `iqr` · `mode` |
| Bivariate | `xs Stats.covariance ys` · `pcovariance` · `correlation` · `linreg` |
| Distributions | `xs Stats.zscores` · `x Stats.normal-pdf {mu, sigma}` · `normal-cdf` |
| Matrix / dataset | `col-means` · `col-variances` · `col-stddevs` · `cov-matrix` · `cor-matrix` · `standardize` · `x Stats.ols ys` |

Every call is receiver-first and ends with `end`:
`receiver Stats.verb args… end`. Unqualified `variance`/`stddev`/
`covariance` are **sample** statistics (n-1); the `p`-prefixed ones are
**population**. Full details are in the [Reference](docs/reference.md).

## For AI coding agents

If an agent will call this library, point it at **[AGENTS.md](AGENTS.md)**
— the exact AQL calling convention, verified idioms, and the common
mistakes to avoid.

To make that guidance available in *another* project that uses this
library, install the bundled skill either way:

- **Copy the skill** — drop
  [`.claude/skills/stats-aql/`](.claude/skills/stats-aql/SKILL.md)
  into that project's `.claude/skills/` (or your `~/.claude/skills/`). It
  loads on demand whenever `Stats` calls appear.
- **Install the plugin** — this repo is also a plugin marketplace:

  ```
  /plugin marketplace add voxgig-aql/stats
  /plugin install stats-aql@voxgig-aql
  ```

Working inside *this* repo, Claude Code picks the guidance up
automatically via `CLAUDE.md` (which imports `AGENTS.md`) and the bundled
skill.

## Project layout

```
stats.aql                  the library (the Stats namespace + Summary type)
AGENTS.md                  agent guide: how to call this library correctly
test/stats_unit_test.aql   example-based unit tests — direct (Test.test)
test/stats_unit_spec.aql   example-based unit tests — declarative spec format
test/stats_prop_test.aql   property-based tests — direct (Test.check-prop)
test/stats_prop_spec.aql   property-based tests — declarative spec format
test/stats_smoke_test.aql  end-to-end smoke run over every public word
docs/                      Diátaxis documentation (above)
dx-report.md               developer-experience notes (current pin: aql @ 12a44e0)
proposals/                 language proposals raised from this module's DX
```

Test files follow a consistent naming convention: `_test.aql` for
direct tests (unit or property), `_spec.aql` for declarative specs (unit
or property).

## Running it

Build the `aql` interpreter, then run any script or test — see
[How-to → Install and run](docs/how-to.md#install-and-run-aql) and
[Run the tests](docs/how-to.md#run-the-tests):

```bash
aql test/stats_unit_test.aql   # unit tests — direct
aql test/stats_unit_spec.aql   # unit tests — declarative spec format
aql test/stats_prop_test.aql   # property tests — direct
aql test/stats_prop_spec.aql   # property tests — declarative spec format
aql test/stats_smoke_test.aql  # end-to-end smoke run
```

A GitHub Actions workflow
([`.github/workflows/test.yml`](.github/workflows/test.yml)) builds aql from a
pinned commit and runs every suite through the interpreter, `aql check`, and
the byte compiler — plus a `consistency` job (agent-skill drift, JSON
manifests, and a pinned-ref guard) — on each push and pull request.

## License

See [LICENSE](LICENSE).
