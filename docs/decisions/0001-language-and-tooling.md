---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0001: Python 3.12+, uv, Typer + Rich, dataclasses + Pydantic at the boundary, Ruff, mypy strict

## Context and Problem Statement

This is a portfolio CLI that coding agents will build over 2–4 weeks. It must install cleanly,
type-check strictly, and use tools that are maintained as of October 2026. The license is MIT, so
every runtime dependency must be compatible with MIT distribution.

## Decision Drivers

- Maintained, stable tools with few moving parts.
- Strict typing, so agents get fast feedback.
- License hygiene for an MIT project.
- Agents follow tool names written in the instructions. The ETH study saw `uv` used 1.6×/instance
  when the file mentioned it, versus < 0.01 when it did not
  ([arXiv 2602.11988](https://arxiv.org/html/2602.11988v1)). Naming one toolchain removes guesswork.

## Considered Options

### Minimum Python
- **≥ 3.12 (chosen).** 3.12 EOL is 2028-10. Gives PEP 695 generics, `TaskGroup` and `asyncio.timeout`.
  3.10 reached EOL on 2026-10-01 ([devguide](https://devguide.python.org/versions/)).
- ≥ 3.11: this is the hard floor from Hypothesis and platformdirs, but 3.11 reaches EOL in 2027-10.
- ≥ 3.13: drops an in-support version for no benefit we need.

### Project management
- **uv (chosen)**: 0.13.0, Production/Stable. `uv init --package` gives a src layout, `uv_build` and
  `uv.lock`; PEP 735 dev group ([uv docs](https://docs.astral.sh/uv/concepts/projects/init/)).
  One tool for Python install, venv, lock, run and build.
  Con: it moves fast, so pin the version in CI.
- Poetry / Hatch / PDM: viable, but not evaluated in depth. Each needs more tools to cover what uv does alone.
- pip + venv: no cross-platform lockfile.

### CLI framework
- **Typer `~=0.27` (chosen)**: actively released (0.27.3, 2026-10-06), type-hint driven, Rich help.
  Con: it is 0.x/Beta, 0.27.0 had a breaking metavar change, and it vendored Click in 0.26
  ([Typer release notes](https://typer.tiangolo.com/release-notes/)). Mitigation: a compatible-release pin and exit-code tests.
- Click 8.5: stable, more boilerplate.
- Cyclopts 5: credible, but Apache-2.0 and less widely used.
- argparse: no dependency, no Rich help or typed parsing.

### Data models
- **Frozen stdlib dataclasses internally + Pydantic v2 only for JSON output (chosen).** Dataclasses
  keep the pure classifier free of third-party types. Pydantic gives `model_json_schema()` (Draft
  2020-12), so the `--json` contract can be committed and checked
  ([Pydantic JSON Schema](https://pydantic.dev/docs/validation/latest/concepts/json_schema/)).
  Con: there are two parallel type sets, so a mapping layer (`report/json_models.py`) is needed.
- Pydantic everywhere: validation overhead and coupling inside hot, pure code.
- attrs / msgspec: msgspec is Beta. attrs adds a dependency without a schema generator.

### Lint, format and type checking
- **Ruff for lint + format, with rule `I` enabled (chosen).** The formatter does not sort imports, so the
  `I` rules are needed ([Ruff formatter](https://docs.astral.sh/ruff/formatter/)).
- **mypy strict as the CI gate (chosen)**: 2.4.0, stable, with the Pydantic plugin.
- pyright strict: equally valid. We pick one gate to avoid conflicting diagnostics.
- ty: 0.0.86, still Beta ([PyPI ty](https://pypi.org/pypi/ty/json); [Astral blog](https://astral.sh/blog/ty)).
  Optional, non-blocking CI job with an exact pin.

### Other runtime libraries
- `platformdirs` (cache and config paths), `dnspython` (Team Cymru TXT, PTR), `maxminddb` (MMDB reader), `rich`.
- Stdlib instead of dependencies: `sqlite3` (not diskcache, whose last release was 2023-08-31), `urllib.request`
  (not httpx, see [ADR-0004](0004-ip-to-asn-source.md)), `tomllib`.

## Decision Outcome

Chosen: Python ≥ 3.12, uv (src layout, `uv.lock` committed, `dev` dependency group), Typer `~=0.27` +
Rich, frozen dataclasses internally with Pydantic v2 at the JSON boundary, Ruff (lint+format, `I`
enabled; suggested selection `E,F,W,I,B,UP,SIM,RUF,ASYNC,S,PT`, line length 100), mypy `strict = true`
with `plugins = ["pydantic.mypy"]`, ty optional. Tests use pytest + pytest-cov + Hypothesis + syrupy
([ADR-0009](0009-testing-strategy.md)). License: MIT.

License check of runtime dependencies (from PyPI JSON): typer MIT, rich MIT, pydantic MIT,
platformdirs MIT. dnspython and maxminddb are believed to be ISC and Apache-2.0 respectively, but
were **not** checked in the research. Verify them in M0. Both are compatible with MIT distribution.
Hypothesis (MPL-2.0) is dev-only and not distributed.

### Consequences

- Good: one command set for agents (`uv sync`, `uv run …`). Strict types catch interface drift
  between layers. The JSON contract is machine-checkable.
- Bad: Typer 0.x may break on minor releases, so pin it and test exit codes. Two model layers need a
  mapping. Ruff `S` rules flag subprocess use, which needs a justified `# noqa: S603` at the one call site.
- Usage errors in Click and Typer exit with code 2 by default. We remap them ([cli.md](../spec/cli.md#exit-codes)).

### Confirmation

CI runs `uv sync --locked`, `ruff check`, `ruff format --check`, `mypy src` and `pytest` on 3.12, 3.13
and 3.14. `pyproject.toml` contains `requires-python = ">=3.12"` and `typer~=0.27`.

## More Information

[python_tooling research](../research.md#python-tooling). Revisit when ty reaches stable.
