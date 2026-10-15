# AGENTS.md

Smart Path Analyzer (`spa`, import package `smart_path_analyzer`) is a Python CLI. It traces to a
target many times using Paris traceroute (a fixed flow tuple, so ECMP cannot fake path changes). It
adds ASN, AS name, IXP and low-confidence location data to each hop. Then it decides where a problem
really is. Loss or latency that shows up at one hop but does not persist to later hops and the
destination is ICMP deprioritization, not a problem. Scope is defined in `docs/roadmap.md`: anything
not listed there is out of scope.

## Commands

```bash
uv sync                                   # install runtime + dev deps from uv.lock
uv run pytest                             # unit/property/snapshot tests (lab + live excluded by default)
uv run ruff check --fix                   # lint (includes import sorting, rule I)
uv run ruff format                        # format
uv run mypy src                           # strict type check — the CI gate
uv run spa --help                         # run the CLI
uv run spa analyze tests/fixtures/recordings/<name>.spa.json   # replay a recorded run, no root needed
uv run pytest --snapshot-update           # ONLY after reviewing every snapshot diff
sudo -E uv run pytest -m lab              # Linux netns lab, needs root; never in CI
```

Add dependencies with `uv add <pkg>` / `uv add --dev <pkg>` (never edit `uv.lock` by hand).

## Repo map (planned)

```
src/smart_path_analyzer/
  cli.py            Typer app; the only place that maps exceptions to exit codes
  models.py         frozen dataclasses + enums shared by all layers
  config.py         TOML config + env vars -> Settings/Thresholds
  probe/            backends/ (scamper, replay), parse/ (scamper JSON), runner, recording
  enrich/           special ranges, iptoasn, Team Cymru, PeeringDB, DB-IP geo, rDNS
  analyze/          stats + classifier: PURE functions, no I/O
  report/           Rich renderer, Pydantic JSON models (schema v1)
  store/            sqlite cache, platformdirs paths
tests/              unit/ property/ snapshot/ e2e/ live/ lab/ fixtures/ builders.py
schema/             committed JSON Schema for --json output
docs/               specs, ADRs, roadmap (see index below)
```

## Hard rules

- IMPORTANT: Never invent scamper CLI flags, JSON field names or output shapes. Use only what
  `docs/spec/probing.md` marks as verified, or what the recorded fixtures in
  `tests/fixtures/scamper/` show. If something is not verified, stop and say so.
  <!-- Why: the research could not run scamper; flags beyond -P/-q/-w/-W are unconfirmed (ADR-0002, M0). -->
- `src/smart_path_analyzer/analyze/` must stay pure. No I/O, no network, no clock, no randomness,
  and no imports from `probe`, `enrich`, `store`, `report` or `cli`. It takes data and returns data.
- Unit and property tests never touch the network, raw sockets or the real cache dir. Use
  `ReplayBackend`, fixture datasets and `tmp_path`.
- Do not change a threshold default, verdict name, exit code or JSON field without asking first.
  Those live in `docs/spec/classifier.md`, `docs/spec/cli.md` and `docs/spec/data-model.md`.
- Before adding a runtime dependency, ask. Then record it in an ADR (new, or a superseding one).
  <!-- Why: license hygiene. The project is MIT; GPL code may only be reached through a process boundary (ADR-0002). -->
- Never import `scapy` or the `scamper` Python module. scamper is driven only as a subprocess,
  using `asyncio.create_subprocess_exec` with an argv list and never a shell.
- Do not contradict an accepted ADR. To change a decision, write a new ADR that supersedes it.
- Never commit real user source addresses in fixtures. Follow the scrub step in `docs/spec/probing.md`.

## Workflow

1. Find the current milestone in `docs/roadmap.md`. Work on one milestone at a time.
2. Read the spec and ADRs that the milestone links to before writing code.
3. For changes that touch more than one module, write a short plan first (files, interfaces,
   tests) and check it against the spec.
4. Write or extend the tests together with the code. Classifier behavior must match the worked
   examples in `docs/spec/classifier.md`.
5. Before saying you are done, run `uv run ruff check --fix`, `uv run ruff format`,
   `uv run mypy src` and `uv run pytest`. All must pass. Show the output.
6. When behavior changes, update the spec in the same change. Tick the acceptance boxes in
   `docs/roadmap.md` only when the listed commands actually pass.

## Gotchas

- Click/Typer exit with status 2 on usage errors. We reserve 2 for "inconclusive", and usage errors
  must exit 3. See `docs/spec/cli.md`.
- `ipaddress.is_private` changed in Python 3.13. Use `enrich/special.py` instead.
- With `--json`, stdout contains only JSON. Progress and warnings go to stderr.
- On a subprocess timeout, call `proc.kill()` and then `await proc.wait()`. Cancelling
  `wait_for` does not kill the child.

## Docs index

- `docs/architecture.md`: components, data flow, module layout, interfaces, errors, config.
- `docs/spec/classifier.md`: the algorithm, thresholds, worked examples, verdict wording.
- `docs/spec/data-model.md`: dataclasses, units, the JSON output schema v1.
- `docs/spec/probing.md`: scamper integration, rounds, privileges, M0 checklist, fixture recording.
- `docs/spec/enrichment.md`: data sources, licenses, rate limits, cache TTLs, precedence.
- `docs/spec/cli.md`: commands, flags, defaults, output formats, exit codes.
- `docs/testing.md`: test layers, fixtures, Hypothesis properties, snapshots, netns lab, CI.
- `docs/roadmap.md`: milestones M0 to M6 with acceptance commands; definition of done.
- `docs/decisions/README.md`: ADR index (why each choice was made).
- `docs/research.md`: sources and the "verify before relying" list.
- `docs/glossary.md`: networking and statistics terms.
