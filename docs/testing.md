# Testing

Rationale: [ADR-0009](decisions/0009-testing-strategy.md). The test oracle comes from
[loss_attribution.md Q6](research.md#loss-attribution-and-statistics): forwarding impairments must
be attributed, and ICMP-only impairments must be dismissed.

## Layers

| Layer | Where | Runs in CI | Needs | What it proves |
|---|---|---|---|---|
| Unit | `tests/unit/` | yes | nothing | stats math, special ranges, parsers, cache TTLs, argv building, CLI exit codes |
| Worked examples | `tests/unit/analyze/test_worked_examples.py` | yes | nothing | classifier matches [classifier.md §9](spec/classifier.md#9-worked-examples-acceptance-tests-for-m2) E1–E12 |
| Property | `tests/property/` | yes | Hypothesis | classifier invariants over generated runs |
| Snapshot | `tests/snapshot/` | yes | syrupy | JSON report, Rich rendering, JSON Schema, scamper argv |
| Replay end-to-end | `tests/e2e/` | yes | recordings | `spa analyze <fixture>` through the real CLI (Typer `CliRunner`) |
| Live smoke | `tests/live/` (`@pytest.mark.live`) | no | scamper + root + Internet | `spa trace` runs against a public target |
| Lab | `tests/lab/` (`@pytest.mark.lab`) | no | Linux, root, iproute2, iptables, tc, scamper | the 12 acceptance scenarios with real packets |

Most tests are unit and property tests. There are a few dozen snapshot and replay tests, and lab
runs happen by hand before a release.

`pyproject.toml` (planned):

```toml
[tool.pytest.ini_options]
addopts = "-ra --strict-markers -m 'not lab and not live' --cov=smart_path_analyzer --cov-report=term-missing"
markers = [
  "lab: Linux netns lab, needs root (sudo -E uv run pytest -m lab)",
  "live: real Internet probing with scamper, needs root",
]
testpaths = ["tests"]
```

To select lab tests explicitly, pass `-m lab`; it overrides the default `-m` in `addopts`.
Coverage gate: `--cov-fail-under=90` for `analyze/` (checked in CI with a separate `coverage report --include`
step), 80 overall.

## Rules

- No network, no raw sockets, no real cache dir in unit, property, snapshot or e2e tests. Use
  `tmp_path` for caches, a `FakeDnsResolver`, a `FakeClock`, and `ReplayBackend`.
  `tests/conftest.py` sets an autouse fixture that blocks `socket.socket.connect` to anything other
  than loopback (`pytest-socket` is **not** a dependency; a 10-line monkeypatch is enough).
- Never hand-write scamper JSON. Parser tests use only committed real output from M0 and later
  ([probing.md](spec/probing.md#recording-fixtures)). Classifier tests build `TraceRun` objects
  directly with the synthetic builder, so they do not depend on the scamper format.

## Fixtures

```text
tests/fixtures/
  scamper/            <case>.jsonl + <case>.md       raw single-round scamper output (M0+), never hand-written
  recordings/         <scenario>.spa.json            full runs (scrubbed), replayed by e2e tests
  datasets/           tiny iptoasn TSV, tiny PeeringDB JSON, tiny test MMDB  (enrichment unit tests)
  lab/                recordings captured from lab scenarios (optional, for regression)
```

Tiny datasets must use only documentation prefixes (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24,
2001:db8::/32) and private ASNs (64496–64511 documentation range, 64512+). Exception: a few real ASN
names (AS3356 Lumen, AS7922 Comcast) may appear as *names* in synthetic enrichment, to match the
wording examples. The test MMDB is built with a test helper: writing MMDB needs a writer library.
If none is acceptable as a dev dependency, mock the `maxminddb` reader behind `GeoReader` instead
(decide in M4, record in ADR-0006).

### Synthetic builder

`tests/builders.py` provides `make_run(...)`, a builder for `TraceRun` with ground truth:

```python
run = make_run(
    hops=[hop(ip="198.51.100.1", rtt=1.0), hop(ip="198.51.100.2", rtt=5.0, icmp_drop=0.3, pattern="alternate"),
          hop(ip="198.51.100.3", rtt=9.0, link_loss=0.06), hop(ip="203.0.113.10", rtt=12.0, dest=True)],
    rounds=100, seed=7,
)
```

Model, per loss_attribution Q6: `link_loss` on hop k drops the probe for **all TTLs ≥ k in that
round** (it propagates). `icmp_drop` drops only hop k's own reply (it does not propagate).
`pattern` is `"bernoulli"`, `"alternate"` (token bucket-like) or `"burst"`. `slow_path_ms` adds delay to
hop k only, `link_delay_ms` adds delay to hops ≥ k, and `mpls_flat=(a, b)` flattens min RTTs of hops a..b.
`ecmp_switch_at=(ttl, round, new_ip)` simulates a path change. The builder is deterministic for a
given `seed`. Use `random.Random(seed)`, never global random. Draw exactly one uniform number per
(round, hop, impairment) and drop when `u < p`. Losses are then coupled across parameter values:
raising p with the same seed only adds losses (property P12 relies on this).

## Hypothesis properties

In `tests/property/test_classifier_properties.py`. Strategies generate builder parameters
(5–20 hops, 20–300 rounds, loss and delay parameters) rather than raw dataclasses, so generated
runs are physically consistent. Use `@settings(deadline=None, max_examples=200)` in CI and
`derandomize=True`, so CI is reproducible.

| ID | Property |
|---|---|
| P1 | Loss injected **only** as `icmp_drop` at intermediate hops (no `link_loss`) never gives `LOSS_STARTS_AT`. |
| P2 | A hop with zero observed loss is never the onset. If no `icmp_drop` is injected, a `LOSS_STARTS_AT` onset (`location.to_hop`) is never before the first hop with injected `link_loss`. (With upstream `icmp_drop` and an unlucky low destination sample, the onset may legitimately fall back to a noisy upstream hop, flagged by `PATH_START`/`ONSET_RANGE`.) |
| P3 | When the verdict is `LOSS_STARTS_AT` or `LATENCY_STARTS_AT`, every responding hop from `location.to_hop` through the destination shows the degradation (loss not significantly below the destination, or median ≥ the persistence floor). |
| P4 | `classify` is deterministic: the same input twice gives equal results, and shuffling the order of replies inside a round does not change the result. |
| P5 | With no injected delay, adding more `icmp_drop` at a hop that already has `ICMP_DEPRIORITIZED` never changes the verdict code, as long as the hop keeps ≥ 20% replies and ≥ `LATENCY_MIN_SAMPLES` replies (below that it becomes neutral or latency-ineligible, which is a different case). |
| P6 | Fewer than `MIN_ROUNDS_ABSOLUTE` completed rounds always gives `INSUFFICIENT_DATA`. |
| P7 | A destination with zero replies always gives `DESTINATION_UNREACHABLE`, unless P6 applies or a path change was detected. |
| P8 | Exit code mapping is total, and `verdict.exit_code` in JSON equals `exit_code_for(verdict)`. |
| P9 | `wilson(x, n)`: `0 ≤ lo ≤ x/n ≤ hi ≤ 1`, and the width decreases as n grows at fixed p̂. |
| P10 | With empty enrichment, `slow_path_ms` at a single intermediate hop (no `link_delay_ms`) never gives `LATENCY_STARTS_AT`. |
| P11 | An empty enrichment mapping never changes the verdict code compared to full enrichment, except for the `DETOUR` rule (which needs geo). |
| P12 | Monotonicity: increasing `link_loss` at a hop, holding the seed, never moves the verdict from `LOSS_STARTS_AT` to `NO_PROBLEM`. |

Statistical flakiness: with random generation, rare sampling extremes are possible (for example
`link_loss = 0.03` with 100 rounds may observe 1 loss). Properties are phrased so they hold for
**every** sample (P1, P2, P4–P11), or they assume parameters with a clear margin (P12: compare
only `link_loss ≥ 0.15` at ≥ 100 rounds). Add every Hypothesis failure as an `@example` regression.

## Snapshot policy (syrupy)

- `JSONSnapshotExtension` for JSON reports, excluding volatile paths (`generated_at`,
  `measurement.started_at`, `measurement.finished_at`, `tool.version`, `data_sources.*.version`).
- Rich output: render to `Console(width=120, color_system=None, record=True)` and snapshot the text.
- `schema/report-v1.schema.json` is compared to `ReportV1.model_json_schema(mode="serialization")`.
  A mismatch fails, which forces a deliberate `schema_version` decision.
- `build_argv` output for each protocol is snapshotted after M0 confirms the flags.
- Update with `uv run pytest --snapshot-update` **only** after reading every diff. A snapshot change in
  a PR must be explained in the PR description. Unused snapshots are deleted by syrupy on update.
  Commit `__snapshots__/`.

## Lab (Linux netns + tc netem + iptables)

**Purpose:** prove the classifier against real packet behavior. Linux routers built from network
namespaces produce real ICMP Time Exceeded messages, `tc netem` creates real forwarding loss and delay,
and `iptables` limits fake ICMP-only loss. Guo & Heidemann validated iptables as a faithful emulator of router ICMP
rate limiting ([PAM 2018](https://www3.isi.edu/~johnh/PAPERS/Guo18a.pdf)).

**All commands below are standard iproute2, tc and iptables usage but are unverified in this
project. Verify before use (M6), then remove this sentence.** Run on a disposable VM, not your
workstation.

Topology: `src — r1 — r2 — r3 — r4 — dst`, with namespaces joined by veth pairs, 10.0.<link>.0/24 per link.

```bash
# tests/lab/setup.sh  (verify before use)
set -euo pipefail
NS=(src r1 r2 r3 r4 dst)
for ns in "${NS[@]}"; do ip netns add "$ns"; ip -n "$ns" link set lo up; done
for i in 0 1 2 3 4; do                      # link i joins NS[i] and NS[i+1]
  a=${NS[$i]}; b=${NS[$((i+1))]}
  ip link add "v${i}a" netns "$a" type veth peer name "v${i}b" netns "$b"
  ip -n "$a" addr add "10.0.$i.1/24" dev "v${i}a"; ip -n "$a" link set "v${i}a" up
  ip -n "$b" addr add "10.0.$i.2/24" dev "v${i}b"; ip -n "$b" link set "v${i}b" up
done
for r in r1 r2 r3 r4; do
  ip netns exec "$r" sysctl -qw net.ipv4.ip_forward=1
  ip netns exec "$r" sysctl -qw net.ipv4.icmp_ratelimit=0      # baseline: no kernel ICMP limiting
done
ip netns exec dst sysctl -qw net.ipv4.icmp_ratelimit=0
# routes: endpoints use a default route; router NS[i] sends everything towards dst via the next
# hop (default) and the subnets behind it back via the previous hop.
ip -n src route add default via 10.0.0.2
ip -n dst route add default via 10.0.4.1
for i in 1 2 3 4; do
  r=${NS[$i]}
  ip -n "$r" route add default via "10.0.$i.2"                 # next hop towards dst
  for j in $(seq 0 $((i-2))); do                                # subnets behind the previous hop
    ip -n "$r" route add "10.0.$j.0/24" via "10.0.$((i-1)).1"
  done
done
```

Target: `10.0.4.2` (dst). Check routes in M6 with `ip netns exec src ping -c1 10.0.4.2` and a manual
scamper trace (expected responders 10.0.0.2, 10.0.1.2, 10.0.2.2, 10.0.3.2, 10.0.4.2). The kernel's
`icmp_ratelimit=0` matters: the default of 1000 ms would itself create intermediate-hop "loss" under
probing ([kernel ip-sysctl](https://docs.kernel.org/networking/ip-sysctl.html)).

Impairment snippets (verify before use):

```bash
# ICMP-only loss at r2 (cosmetic): limit locally generated Time Exceeded
ip netns exec r2 iptables -A OUTPUT -p icmp --icmp-type time-exceeded -m limit --limit 30/minute --limit-burst 5 -j ACCEPT
ip netns exec r2 iptables -A OUTPUT -p icmp --icmp-type time-exceeded -j DROP
# Real forwarding loss on link r2→r3 (egress of r2 towards r3); scenario #3 uses 10%
ip netns exec r2 tc qdisc add dev v2a root netem loss 10%
# Real queueing-like latency on r3 egress towards r4 (jitter keeps the min RTT low, the median rises)
ip netns exec r3 tc qdisc add dev v3a root netem delay 40ms 20ms distribution normal
# Variant 5b: constant delay = propagation-like step (min rises with the median)
ip netns exec r3 tc qdisc add dev v3a root netem delay 40ms
# Return-path-only delay: r3's egress back towards r2
ip netns exec r3 tc qdisc add dev v2b root netem delay 40ms 20ms distribution normal
# Destination ignores the probe type (UDP example)
ip netns exec dst iptables -A INPUT -p udp -j DROP
# Silent router r2
ip netns exec r2 iptables -A OUTPUT -p icmp --icmp-type time-exceeded -j DROP
# Bursty loss
ip netns exec r2 tc qdisc add dev v2a root netem loss gemodel 1% 10%
```

`netem` syntax is from [tc-netem(8)](https://man7.org/linux/man-pages/man8/tc-netem.8.html), and the
iptables `limit` and `--icmp-type time-exceeded` syntax is from [iptables-extensions(8)](https://ipset.netfilter.org/iptables-extensions.man.html).
Whether locally generated ICMP errors go through the OUTPUT chain is an inference in the notes. Verify it.
The cosmetic-latency filter (delay only ICMP type 11 via tc) has no known recipe, so it needs prototyping.
Scenario 6 is optional until one exists.

Rate note: we send about 1 probe per hop per second, so the limit must be below that to trigger
(a 2/second limit would never drop anything). It must also not be so low that the hop becomes
`MOSTLY_SILENT` (reply rate under 10%), which counts as neutral, not deprioritized. `30/minute` with
burst 5 should give about 50% replies in an on/off pattern. Confirm the observed hop loss is between
20% and 80% in M6, and adjust here if not.

Lab tests (`tests/lab/test_scenarios.py`) call setup and impairment helpers, run
`spa trace 10.0.4.2 --json --offline --rounds N` inside the `src` namespace, assert the verdict, and
tear down (`ip netns del` for each) in a `finally` block. Each scenario also saves its recording to
`tests/fixtures/lab/` so it can be replayed in CI later.

## Acceptance scenarios

These come from loss_attribution Q6. Hops in the lab: r1 = 1, r2 = 2, r3 = 3, r4 = 4, dst = 5.
"Rounds" means `--rounds`.

| # | Scenario | Impairment | Rounds | Expected verdict | Expected detail | Exit |
|---|---|---|---|---|---|---|
| 1 | Clean path | none | 100 | `NO_PROBLEM` | confidence high | 0 |
| 2 | ICMP-only loss at r2 | iptables limit on r2 OUTPUT time-exceeded | 100 | `ICMP_DEPRIORITIZATION_ONLY` | hop 2 flagged `ICMP_DEPRIORITIZED` | 0 |
| 3 | Real loss r2→r3 | netem loss 10% on r2 egress | 100 | `LOSS_STARTS_AT` | between hops 2 and 3; location `unknown_as` and confidence medium (lab addresses are private, `--offline`) | 1 |
| 4 | Real loss + upstream rate limiter | #2 on r1 + #3 | 100 | `LOSS_STARTS_AT` | between hops 2 and 3; hop 1 `ICMP_DEPRIORITIZED` | 1 |
| 5 | Real latency step (queueing-like) | netem delay 40ms 20ms normal on r3 egress | 100 | `LATENCY_STARTS_AT` | between hops 3 and 4; caveat `RETURN_PATH` | 1 |
| 5b | Constant delay step (propagation-like) | netem delay 40ms on r3 egress | 100 | `NO_PROBLEM` | finding `PROPAGATION_STEP` at hop 4 (no geo, so distance cannot be ruled out; [classifier.md §5.6](spec/classifier.md#56-latency)) | 0 |
| 6 | Cosmetic latency at r2 (optional) | delay only ICMP type 11 from r2 | 100 | `ICMP_DEPRIORITIZATION_ONLY` | hop 2 `SLOW_PATH_ARTIFACT` | 0 |
| 7 | Return-path-only delay | netem delay 40ms 20ms normal on r3 towards r2 | 100 | `LATENCY_STARTS_AT` | between hops 2 and 3 (r3's replies are delayed on the way back); caveat `RETURN_PATH` | 1 |
| 8 | Destination ignores probes | dst drops the probe protocol | 30 | `DESTINATION_UNREACHABLE` | last answering hop 4 | 2 |
| 9 | Silent router | r2 drops all time-exceeded | 100 | `NO_PROBLEM` | hop 2 `SILENT`, finding `SILENT_HOP` | 0 |
| 10 | Bursty loss | netem gemodel on r2 egress | 300 | `LOSS_STARTS_AT` or `INSUFFICIENT_DATA` | never `NO_PROBLEM` with high confidence; if `LOSS_STARTS_AT`, caveat `BURSTY_CI_OPTIMISTIC` when runs z ≤ −2 | 1 or 2 |
| 11 | ECMP, loss on one branch | Extra namespace r3b parallel to r3; Linux multipath route at r2 (two next hops), loss 10% on the r2→r3b link only | 100 | `LOSS_STARTS_AT` or `NO_PROBLEM`, consistently for one flow; never `PATH_UNSTABLE` | Paris keeps the flow on one branch. Repeat with 2 different `--sport` values to hit both branches | 0 or 1 |
| 12 | Small sample | same as #3 but loss 5% | 10 | `INSUFFICIENT_DATA` | "only 10 rounds" | 2 |

Notes:
- Lab addresses are private and the lab runs `--offline`, so loss onsets have location kind
  `unknown_as`, and `LOSS_STARTS_AT` confidence is capped at medium (#3, #4). The assertions check
  hop numbers, not AS names.
- #3 uses 10% (not 5%) so 100 rounds give a clear margin. P(observing ≤ 4 losses out of 100 at a true
  10%) ≈ 2.4%, and 5/100 is already LOSSY. With 5% true loss, 100 rounds is below 3 losses about 12% of
  the time, which would make the scenario flaky.
- #7 is the "honest wording" check. The tool cannot see the return path, so it must attribute the
  step to where it appears and carry the `RETURN_PATH` caveat.
- #11 needs a multipath hash policy that includes ports (`net.ipv4.fib_multipath_hash_policy=1`,
  verify before use). It checks that a fixed flow means "no fake path changes".
- Lab expectations (exact hop numbers) may need adjusting once the topology is real. Change them
  in this table and in the test together, and explain why in the commit.

## CI

`.github/workflows/ci.yml` (planned), from the [uv GitHub Actions guide](https://docs.astral.sh/uv/guides/integration/github/):

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest]
        python-version: ["3.12", "3.13", "3.14"]
        include:
          - os: macos-latest
            python-version: "3.13"
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@<sha>          # pin to a SHA
      - uses: astral-sh/setup-uv@<sha>        # v9.x, pinned; inputs below
        with:
          version: "0.13.0"
          python-version: ${{ matrix.python-version }}
          enable-cache: true
      - run: uv sync --locked
      - run: uv run ruff check
      - run: uv run ruff format --check
      - run: uv run mypy src
      - run: uv run pytest
  build:                                       # runs-on + checkout + setup-uv as above, abbreviated
    steps:
      - run: uv build
      - run: uvx --from dist/*.whl spa --help   # smoke test the wheel (verify glob works in CI shell)
  ty:                                          # optional, non-blocking; setup abbreviated
    continue-on-error: true
    steps:
      - run: uvx ty@<exact version> check src
```

CI never runs `lab` or `live` tests and never needs scamper or root.
