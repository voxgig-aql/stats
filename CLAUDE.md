# CLAUDE.md

This repository is the `Stats` statistics library, written in boru —
descriptive, inferential, and matrix statistics exposed as the single
`Stats` namespace plus a streaming `Summary` accumulator.

## Using the library

See @AGENTS.md for how to call the `Stats` API correctly from boru — the
calling convention, the full API, copy-paste idioms, and the common
mistakes to avoid. Every example there is verified against the pinned
`boru` build.

## Working on this repository

- A SessionStart hook (`.claude/settings.json` →
  `.claude/hooks/session-start.sh`) builds `boru` from the pinned commit in
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
- `test/divergence/run.sh` runs every suite through all three boru surfaces —
  interpreter, `boru check`, and the byte compiler (`boru --compile`) — and
  asserts none errors or disagrees. It builds its own boru at the pinned ref
  (via a codeload tarball, so it works even where raw `git clone` of boru is
  blocked). See its `README.md`.
- Known boru-runtime gotchas observed with the pinned build are in
  `dx-report.md`. The pinned boru commit (`6185620…`) is single-sourced in
  `.github/workflows/test.yml`'s `BORU_REF`; a CI job fails if the hook,
  `test/divergence/run.sh`, or `api.json` drift from it.
- Forking this repo to start a new boru library? See `TEMPLATE.md` on the
  template branch (this instantiated copy has removed it).
