# CLAUDE.md

This repository is the `Stats` statistics library, written in AQL —
descriptive, inferential, and matrix statistics exposed as the single
`Stats` namespace plus a streaming `Summary` accumulator.

## Using the library

See @AGENTS.md for how to call the `Stats` API correctly from AQL — the
calling convention, the full API, copy-paste idioms, and the common
mistakes to avoid. Every example there is verified against the pinned
`aql` build.

## Working on this repository

- A SessionStart hook (`.claude/settings.json` →
  `.claude/hooks/session-start.sh`) builds `aql` from the pinned commit in
  remote sessions, so a fresh session can run the suites. Locally, build it
  once from source (there is no tagged release and `go install …/aql@latest`
  is blocked by replace directives) — see
  [docs/how-to.md](docs/how-to.md#install-and-run-aql).
- Tests live in `test/`, named `<subject>_<unit|prop>_<test|spec>.aql` plus a
  `stats_smoke_test.aql`: `_test` = imperative (`Test.test`/`Test.check-prop`),
  `_spec` = declarative spec; `unit` = example-based, `prop` = property-based.
  Each assertion-bearing suite ends by asserting `Test.fail-count` is `0` and
  prints `all green`. Exact-valued statistics are asserted directly; the
  irrational ones (stddev, correlation, the normal CDF) are tolerance-checked.
- `test/divergence/run.sh` runs every suite through all three aql surfaces —
  interpreter, `aql check`, and the byte compiler (`aql --compile`) — and
  asserts none errors or disagrees. It builds its own aql at the pinned ref
  (via a codeload tarball, so it works even where raw `git clone` of aql is
  blocked). See its `README.md`.
- Known AQL-runtime gotchas observed with the pinned build are in
  `dx-report.md`. The pinned aql commit is single-sourced in the CI workflow's
  `AQL_REF`; a CI job fails if the hook, `test/divergence/run.sh`, or
  `api.json` drift from it. The workflow currently lives in `ci/test.yml`
  pending promotion to `.github/workflows/` (this session can't push workflow
  files — see `ci/README.md`); the stale `.github/workflows/test.yml` still on
  `main` is superseded by it.
- Forking this repo to start a new AQL library? See `TEMPLATE.md` on the
  template branch (this instantiated copy has removed it).
