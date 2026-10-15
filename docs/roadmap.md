# Roadmap

Plan: about 3 weeks of focused work (M0–M6), with each milestone 1–4 days. Work one milestone at a time.
Tick a box only when its command has actually passed. Every milestone also requires the
**standard gate**:

```bash
uv run ruff check && uv run ruff format --check && uv run mypy src && uv run pytest
```

## Scope

In scope: one vantage point (this machine), one IPv4 target per run, Paris probing via scamper,
repeated rounds, per-hop stats, ASN/AS name/IXP/geo enrichment, the classifier in
[classifier.md](spec/classifier.md), Rich and JSON output, replay of recordings.

Out of scope: multi-vantage monitoring, scheduling or daemons, BGP correlation, web UI, Windows,
MDA/multipath statistics, reverse traceroute. IPv6 is a stretch goal (the data model already allows it).

---

## M0: Spike: verify scamper (1–2 days)

Read: [probing.md](spec/probing.md), [ADR-0002](decisions/0002-probing-backend.md), [ADR-0003](decisions/0003-measurement-strategy.md).

Tasks
- Install scamper on Linux (and macOS if available). Work through the
  [M0 checklist](spec/probing.md#m0-spike-checklist) by hand. No project code is needed beyond a
  throwaway shell script kept in `scripts/m0/` (it may be deleted later).
- Commit the raw outputs to `tests/fixtures/scamper/` with a `.md` note for each.
- Rewrite every UNVERIFIED item in `docs/spec/probing.md`, and every M0-tagged UNVERIFIED item in
  `docs/spec/data-model.md` and `docs/spec/classifier.md`, as verified fact or a decision.
  Update the argv shape and field mapping. If a classifier input (for example `probed_ttls`) cannot be
  derived, write a superseding ADR before M1.
- Create the repo skeleton: `uv init --package smart-path-analyzer`, `requires-python = ">=3.12"`,
  MIT `LICENSE`, `[project.scripts] spa = "smart_path_analyzer.cli:main"`, dev group, Ruff and mypy
  config, an empty `cli.py` with `--version`, and CI workflow ([testing.md](testing.md#ci)).

Acceptance
- [ ] `ls tests/fixtures/scamper/*.jsonl | wc -l` ≥ 5 (clean, silent destination, `*` hops, ICMP-Paris, TCP 443).
- [ ] `grep -c UNVERIFIED docs/spec/probing.md` prints `0`, and `grep -rn UNVERIFIED docs/spec | grep -c M0` prints `0`.
- [ ] `uv run spa --version` prints `0.1.0.dev0`.
- [ ] Standard gate passes (on an empty test suite plus one smoke test). CI is green on 3.12, 3.13 and 3.14.

## M1: Models, parser, runner, recordings (2–3 days)

Read: [data-model.md](spec/data-model.md), [probing.md](spec/probing.md), [architecture.md](architecture.md#interfaces).

Tasks
- `models.py` (all dataclasses and enums), `errors.py`.
- `probe/parse/scamper_json.py` with tests on every M0 fixture.
- `ScamperBackend` (`build_argv`, `preflight`, `run_round` with kill-on-timeout) and `ReplayBackend`.
- `probe/runner.py` (sequential rounds, interval pacing via `Clock`, Ctrl-C, flow check, failed-round abort).
- `probe/recording.py` (write/read `.spa.json`, scrubbing).
- Record 2 real runs (30 rounds) into `tests/fixtures/recordings/`.

Acceptance
- [ ] `uv run pytest tests/unit/probe -q` passes. It includes: a parser test for each fixture, the
      `build_argv` snapshot for each protocol, a timeout test using a fake subprocess that sleeps
      (child killed, round marked TIMEOUT), and a flow-mismatch test (raises `FlowMismatchError`).
- [ ] A test round-trips a recording through `ReplayBackend` (write, then read, equal `TraceRun`).
- [ ] The scrub test passes: a recording never contains the original source IP string.

## M2: Classifier (4 days), the core

Read: [classifier.md](spec/classifier.md) (all), [ADR-0007](decisions/0007-classifier-statistics-and-thresholds.md), [testing.md](testing.md#synthetic-builder).

Tasks
- `analyze/thresholds.py`, `stats.py`, `hops.py`, `path.py`, `attribution.py`, `classifier.py`, `wording.py`.
- `tests/builders.py` synthetic builder.
- Worked examples E1–E12 as tests. Hypothesis properties P1–P12.
- Import-purity test for `analyze/`.

Acceptance
- [ ] `uv run pytest tests/unit/analyze -q` passes, including `test_wilson_reference_table`
      (all rows of the CI table in classifier.md §3 to 2 decimals) and `test_worked_examples` (E1–E12:
      verdict, confidence, location hops and summary string exactly as written in the spec).
- [ ] `uv run pytest tests/property -q` passes with `max_examples=200`.
- [ ] `uv run pytest tests/unit/test_import_rules.py` passes (analyze imports only stdlib and models).
- [ ] `uv run pytest --cov=smart_path_analyzer.analyze --cov-fail-under=90 tests/unit/analyze tests/property` passes.

## M3: ASN enrichment + cache (2–3 days)

Read: [enrichment.md](spec/enrichment.md), [ADR-0004](decisions/0004-ip-to-asn-source.md).

Tasks
- `store/paths.py`, `store/cache.py` (schema, TTLs, injectable clock).
- `enrich/special.py` (table and tests per row), `enrich/download.py`, `enrich/iptoasn.py`
  (convert + bisect), `enrich/cymru.py` (record one real TXT answer per zone as a fixture first),
  and `enrich/service.py` (ASN parts).
- `spa data update --only iptoasn`, `spa data status`.

Acceptance
- [ ] `uv run pytest tests/unit/enrich tests/unit/store -q` passes: special-range table (one address
      per row, including 100.64.0.1 → CGNAT on every Python in the matrix), bisect lookup on the
      tiny dataset (first, last, between-range, v6), TTL expiry with `FakeClock`, and Cymru parsing
      against recorded TXT fixtures.
- [ ] Manual: `uv run spa data update --only iptoasn && uv run spa data status` shows iptoasn with a
      version and age. Running it again prints `skipped: fresh`.
- [ ] Manual: `uv run spa analyze tests/fixtures/recordings/<clean>.spa.json` shows AS numbers and names.

## M4: IXP + geo enrichment (2–3 days)

Read: [enrichment.md](spec/enrichment.md), [ADR-0005](decisions/0005-ixp-detection-peeringdb.md), [ADR-0006](decisions/0006-geolocation.md).

Tasks
- Verify PeeringDB field names against https://www.peeringdb.com/apidocs/ and DB-IP MMDB fields and
  URL. Update enrichment.md first.
- `enrich/peeringdb.py` (sync with 3 s spacing, optional API key), `enrich/geo.py`, `enrich/rdns.py` + `iata.csv`.
- Attribution logic is already in M2. Wire in real `HopEnrichment` now.

Acceptance
- [ ] `uv run pytest tests/unit/enrich -q` passes: exact netixlan match overrides origin ASN; a prefix-only
      match gives no member; the sync issues ≤ 5 requests at least 2 s apart (asserted with a fake
      opener and `FakeClock`); the API key header is sent only when the env var is set; an IATA hint
      parses `ae-1.r01.chi01.example.net` → Chicago; an RTT-implausible geo gets `GEO_IMPLAUSIBLE`.
- [ ] Manual: `uv run spa data update` completes within the anonymous limit, and `spa data status`
      shows the DB-IP attribution text.

## M5: Reports + CLI (2–3 days)

Read: [cli.md](spec/cli.md), [data-model.md](spec/data-model.md#json-output-schema-v1-spa-trace---json-spa-analyze---json), [ADR-0008](decisions/0008-output-and-exit-codes.md).

Tasks
- `report/json_models.py` + committed `schema/report-v1.schema.json`. `report/rich_report.py`.
- Full `cli.py`: `trace`, `analyze`, `data update`, `data status`, flags, config file, exit-code mapping.
- E2E replay tests and snapshots.

Acceptance
- [ ] `uv run pytest tests/snapshot tests/e2e -q` passes: JSON and Rich snapshots for at least 4 verdicts,
      the schema file matches the generated one, and `verdict.exit_code` equals the process exit code.
- [ ] `uv run spa analyze tests/fixtures/recordings/<clean>.spa.json --json | python -m json.tool` succeeds,
      and stdout contains nothing but JSON.
- [ ] `uv run spa trace example.net --rounds abc; echo $?` prints `3`.
- [ ] `uv run spa analyze does-not-exist.json; echo $?` prints `3`.
- [ ] Manual (as non-root on Linux): `uv run spa trace example.net; echo $?` prints the privilege message and `4`.
- [ ] Manual (root): `sudo -E uv run spa trace <public target> --quick` completes in under 90 s and
      prints a verdict line and the `QUICK_MODE` caveat.

## M6: Lab validation, docs, release (3–4 days)

Read: [testing.md](testing.md#lab-linux-netns--tc-netem--iptables).

Tasks
- Make `tests/lab/setup.sh` work and remove the "verify before use" notes that are now verified.
- Implement scenarios 1–12 (6 optional). Tune only those thresholds marked Suggested, and only with
  approval. Record any change in a superseding ADR.
- Save lab recordings to `tests/fixtures/lab/` and add them to e2e replay tests.
- README: real sample output, install steps verified on Ubuntu and macOS, and a Docker image
  (`Dockerfile`) verified with `--cap-add=NET_RAW --network host`.

Acceptance
- [ ] `sudo -E uv run pytest -m lab -v` passes all mandatory scenarios (1–5, 5b, 7–12) on a Linux VM.
      Paste the output into the release PR.
- [ ] `uv run pytest tests/e2e -q` replays every lab recording with the expected verdict (this runs in CI).
- [ ] `uv build && uvx --from dist/smart_path_analyzer-0.1.0-py3-none-any.whl spa --help` works.
- [ ] `docker run --rm --cap-add=NET_RAW --network host spa:0.1.0 spa trace <target> --quick` works on Linux.

---

## Definition of done (v0.1.0)

- [ ] All milestone boxes above are ticked, and CI is green on 3.12, 3.13, 3.14 (ubuntu) and 3.13 (macos).
- [ ] `spa trace` gives the expected verdict for every mandatory lab scenario.
- [ ] Every verdict code in [classifier.md](spec/classifier.md) is covered by a worked-example test, a
      snapshot and at least one e2e replay.
- [ ] No "UNVERIFIED" marker remains in `docs/spec/probing.md`. Any that remain elsewhere are
      listed in [research.md](research.md#known-unknowns--verify-before-relying) as accepted risks.
- [ ] README attribution (DB-IP CC BY 4.0, iptoasn PDDL, PeeringDB) appears in output and `spa data status`.
- [ ] `uv run mypy src` is strict and clean, with no unexplained `type: ignore`.
- [ ] Version `0.1.0` is tagged. CHANGELOG entry written.

## Stretch goals

Only after v0.1.0. Each needs an ADR.

- **IPv6 targets.** The models already use `IPAddress`. Needs scamper v6 fixtures, the IPv6 special-range
  review (2002::/16, 64:ff9b::/96), Cymru `origin6`, and lab scenarios with `icmp_ratelimit` for v6
  (default not confirmed in the research).
- **trippy backend** (`TrippyBackend`, subprocess `--mode json`). Paris is UDP only, and the JSON
  schema is unverified ([Trippy CLI](https://trippy.rs/reference/cli/)). It produces per-hop summaries,
  not per-probe data, so it needs an adapter decision.
- **MDA discovery** with scamper `tracelb`, then one fixed-flow series per discovered branch
  ([Vermeulen et al.](https://ar5iv.arxiv.org/html/1809.10070)).
- **Per-segment analysis** for `PATH_UNSTABLE` runs (classify each stable segment separately).
- **Cycle-aligned paired loss analysis** (loss_attribution Q4) and block bootstrap for bursty loss.
- **CAIDA Hoiho** rDNS geolocation; bgp.tools `asns.csv` or CAIDA AS2Org for better AS names.
