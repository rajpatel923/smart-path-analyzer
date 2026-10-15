---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0003: N repeated single-attempt Paris rounds on one fixed flow, about 1 per second, default 100

## Context and Problem Statement

A single trace cannot estimate loss. We need many probes per hop, spaced so that we do not trigger
the ICMP rate limiting we are trying to tell apart from real loss. The probes must also stay on one
flow so that ECMP does not mix routers within a hop. How should probes be grouped and paced, how
many should be sent, and how do we notice that the path changed during a run?

## Decision Drivers

- Statistical power. Wilson 95%: 0/10 lost is still consistent with up to 27.8% true loss,
  0/100 with 3.7%, and 0/300 with 1.26% ([NIST](https://itl.nist.gov/div898/handbook/prc/section2/prc241.htm);
  table in [classifier.md](../spec/classifier.md#3-statistics-analyzestatspy)).
- Rate-limit hygiene. Linux spaces rate-limited ICMP errors by 1000 ms by default
  ([ip-sysctl](https://docs.kernel.org/networking/ip-sysctl.html)), IPv6 routers are often around 10 Hz
  with a bucket of about 20 ([IETF 99 MAPRG](https://datatracker.ietf.org/meeting/99/materials/slides-99-maprg-rate-limiting-of-ipv6-traceroutes-is-widespread-measurements-and-mitigations-02.pdf)),
  and the tool can create the artifacts it reports
  ([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf)).
- Run time a user will tolerate: 1–2 minutes by default.
- Same raw format for live and replay.

## Considered Options

1. **N rounds, each one full Paris trace with one attempt per TTL, same flow tuple, ~1 s apart (chosen)**
2. One scamper trace with `attempts=N, all_attempts=True` (N probes per TTL inside one trace)
3. TTL-limited pings per hop (scamper `ping` with a TTL)
4. mtr-style: rely on an external tool's per-hop summary statistics

## Pros and Cons of the Options

### 1. Repeated rounds
- Good: matches the 1-second cycle convention of mtr, trippy and scamper ping. Every hop gets
  exactly one probe per round, and losses can be compared by round across hops (useful for path
  changes and for future paired analysis).
- Good: each round's raw output is self-contained, so recording and replay are simple.
- Bad: process spawn per round. Duration is about rounds × 1 s, plus extra time on silent tails.
- Bad: relies on fixed sport and dport being honored across separate processes (M0 checklist).

### 2. attempts=N inside one trace
- Good: a single process, and scamper handles timing.
- Bad: N probes per TTL are sent back to back with a wait in between, so pacing per hop is less
  controllable. One long trace is all-or-nothing on failure. Whether `all_attempts` yields N
  records per TTL was not verified ([probing_engine Q7 gap](../research.md#known-unknowns--verify-before-relying)).

### 3. TTL-limited pings
- Good: natural loss and RTT statistics per hop.
- Bad: the ping TTL option was not confirmed. A different probe type from the trace may hash differently.
  It needs one series per hop, which costs more.

### 4. External summaries (mtr/trippy)
- Bad: no per-probe data (no runs test, no path-change detection), and mtr is not Paris.

## Decision Outcome

Chosen: **option 1**.

- Default `--rounds 100`. `--quick` = 30 rounds plus a warning that only large loss can be called
  (0/30 can only rule out loss above 11.4%). Values below 20 always give `INSUFFICIENT_DATA`.
  <!-- 100 resolves ≥2% loss; 300 is needed for ~1% (loss_attribution Q4). 100 keeps the default near 2 minutes. -->
- Pacing: rounds start at least `--interval` (default 1.0 s) apart and never overlap. One probe per
  hop per round.
- Per-hop loss = rounds where a probe was sent at that TTL but got no reply ÷ rounds where a probe
  was sent at that TTL ([classifier.md §4](../spec/classifier.md#4-per-hop-aggregation-analyzehopspy)).
- The flow tuple is chosen once per run: random sport in 49152–65535; dport 33435 (UDP) or 443 (TCP).
  The parser checks that every round reports the same tuple.
- Path changes: every responder per TTL per round is kept. The classifier reports `PATH_UNSTABLE`
  instead of merging stats ([classifier.md §5.2](../spec/classifier.md#52-path-change-analyzepathpy)).
- Async orchestration: `asyncio.create_subprocess_exec` + `communicate()` inside `wait_for`. On
  timeout, explicit `kill()` then `wait()` ([asyncio docs](https://docs.python.org/3/library/asyncio-subprocess.html)).

### Consequences

- Good: clean per-round data. Statistics are honest about sample size. Replays are exact.
- Bad: a 100-round run takes about 2 minutes. Detecting 1% loss needs `--rounds 300` (about 5 minutes).
- Bad: concurrent multi-target runs are deliberately unsupported (they would raise ICMP load on shared hops).

### Confirmation

- Runner tests with `FakeClock` assert round start spacing ≥ interval and no overlap.
- A test asserts that `--quick` adds `QUICK_MODE` and that < 20 rounds gives `INSUFFICIENT_DATA` (property P6).
- The M0 checklist confirms that the same sport is honored across processes.

## More Information

[probing_engine Q6](../research.md#probing-and-paris-traceroute), [loss_attribution Q4](../research.md#loss-attribution-and-statistics).
