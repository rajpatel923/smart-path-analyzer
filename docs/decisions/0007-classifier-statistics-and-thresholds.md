---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0007: Persistence-based classifier with Wilson/Newcombe statistics and named, configurable thresholds

## Context and Problem Statement

The product promise is to say where a problem really is, and to say "no real problem" when an
intermediate hop only looks bad. Routers generate ICMP Time Exceeded on a low-priority, rate-limited
slow path, so loss or delay seen only in a hop's own replies says nothing about forwarded traffic
([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf);
[Cloudflare](https://www.cloudflare.com/learning/network-layer/what-is-mtr/)). Practitioner rule:
"Unless packet loss or increased RTT is seen on every hop between a given hop and the end of the trace,
it is not a problem" ([APNIC/Lavin](https://blog.apnic.net/2022/03/28/how-to-properly-interpret-a-traceroute-or-mtr/)).
How do we turn that rule into a deterministic, testable algorithm that is honest about uncertainty?

## Decision Drivers

- No false alarms from ICMP deprioritization (the main thing that sets this tool apart).
- Honest abstention when samples are small. FADER's explicit "cannot tell" class was "never wrong"
  when it did answer ([Guo & Heidemann](https://www3.isi.edu/~johnh/PAPERS/Guo18a.pdf)).
- A pure function, so it can be unit- and property-tested.
- Known failure modes must be surfaced, not hidden: return-path asymmetry, MPLS, silent
  destinations, ECMP, AS-boundary ambiguity.

## Considered Options

### Loss decision statistic
1. **Wilson score interval per hop, Newcombe difference interval between hops (chosen)**. NIST
   recommends Wilson for "virtually all combinations of n and p", and its lower limit is never negative
   ([NIST](https://itl.nist.gov/div898/handbook/prc/section2/prc241.htm)). Newcombe reuses Wilson and is closed-form.
2. Normal (Wald) interval: gives impossible values for small n or few losses (NIST).
3. Fisher exact test between hops: valid, but needs a p-value implementation. It is offered in the notes as an alternative.
4. Raw percentages with fixed cutoffs (mtr-style): 10 probes cannot distinguish 0% from 28%.

### Latency statistic
1. **Min (propagation floor) and median with an order-statistic Wilson CI (chosen)**, as used by
   Fontugne et al. ([arXiv 1605.04784](https://arxiv.org/pdf/1605.04784)). Mean, max and stdev are dominated by
   slow-path scheduling and never drive a verdict.
2. Mean ± stdev: Fontugne's example has stdev 12.2 ms against a mean of 4.8 ms, while medians stayed between 5.2 and 5.4 ms.

### Rate-limit fingerprint
1. **Wald–Wolfowitz runs test on the per-round reply sequence (chosen)** as a cheap stand-in for
   Guo & Heidemann's observation that rate limiting alternates while congestion loss is random. It is
   evidence only. The deprioritization verdict rests on the persistence test, not on the pattern.
2. A full FADER-style probing-rate sweep: out of scope (changes the measurement).

### Attribution granularity
1. **Link between last clean hop and first degraded hop, with AS-boundary wording "between AS X and AS Y" (chosen)**.
   Interconnect addresses usually come from one side, and naive IP-to-AS mapping puts borders in the wrong place
   ([bdrmap](https://users.caida.org/~mjl/pubs/bdrmap.pdf)).
2. Blame the router at the first degraded hop: overclaims.

### Latency steps that persist but match distance
1. **Report a latency problem only for queueing-like steps (min flat, median up), or propagation-like
   steps that exceed a physics allowance computed from medium-confidence geo (chosen)**. Otherwise
   use an informational `PROPAGATION_STEP` finding.
2. Report every persistent step: an ocean crossing would be called a fault (APNIC/Lavin).

## Decision Outcome

Chosen: the algorithm in [classifier.md](../spec/classifier.md). Its key commitments:

- **Persistence rule**: loss counts as real only if the destination is LOSSY. The onset is the first
  confirmed-degraded hop after the last hop that is significantly cleaner than the destination (the
  "last clean hop"). Hops with significantly more loss than the destination are `ICMP_DEPRIORITIZED`.
  A latency step counts as real only if ≥ 80% of it persists at every later eligible hop through the destination.
- **Abstain** (`INSUFFICIENT_DATA`) when fewer than 20 rounds completed, or when destination loss is
  neither clearly clean nor clearly lossy.
- **Silent destination** gives `DESTINATION_UNREACHABLE`: end-to-end loss cannot be assessed, so we say so.
- **Silent and mostly-silent hops are neutral.** Path changes give `PATH_UNSTABLE` instead of merged stats.
- **Every latency verdict carries the `RETURN_PATH` caveat.** AS-boundary loss carries `AS_BOUNDARY_SIDE`.
  MPLS runs (repeating latency, RFC 4950 labels) take attribution to the segment level.
- Verdict enum: `NO_PROBLEM`, `ICMP_DEPRIORITIZATION_ONLY`, `LOSS_STARTS_AT`, `LATENCY_STARTS_AT`,
  `DESTINATION_UNREACHABLE`, `PATH_UNSTABLE`, `INSUFFICIENT_DATA`. Confidence is `high`/`medium`/`low`
  with explicit caps, or null for the three inconclusive verdicts.
- **All numbers are named fields of `Thresholds`**, configurable under `[thresholds]` and echoed in
  the report when overridden. Each is marked Sourced or Suggested in the spec. The suggested ones
  (for example `LOSS_PRACTICAL = 0.02`, `LATENCY_STEP_MIN_MS = 10`, `ROUTE_INFLATION = 2.0`) are design
  choices with no published standard behind them (research gaps 16–17), and M6 lab results may revise them.

### Consequences

- Good: dismissing cosmetic hop loss is a tested property (P1), not a hope. Small runs abstain instead of guessing.
- Bad: CIs assume independent losses. Bursty loss makes them optimistic, so we add the
  `BURSTY_CI_OPTIMISTIC` caveat and a confidence cap. Block bootstrap is a stretch goal.
- Bad: one-sided measurement can never rule out the return path, so latency verdicts are capped at medium.
- Bad: propagation-like steps without rDNS geo are never called problems (may miss real detours).
- Changing a default threshold changes product behavior. It needs approval and a superseding ADR.

### Confirmation

Worked examples E1–E12 as tests. Hypothesis properties P1–P12. The Wilson reference table to 2 decimals.
An import-purity test for `analyze/`. Lab scenarios 1–12 ([testing.md](../testing.md#acceptance-scenarios)).

## More Information

[loss_attribution research](../research.md#loss-attribution-and-statistics).
