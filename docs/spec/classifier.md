# Spec: classifier

Status: accepted for v0.1. Rationale: [ADR-0007](../decisions/0007-classifier-statistics-and-thresholds.md).
Inputs and outputs are defined in [data-model.md](data-model.md). Exit codes are in [cli.md](../spec/cli.md#exit-codes).

The classifier is the core of the project. It decides whether there is a real problem and where
it starts. Loss or latency that appears at a hop but does not persist downstream is classified
as ICMP slow-path behavior, not a forwarding problem.

## 1. Contract

```python
# src/smart_path_analyzer/analyze/classifier.py
def classify(
    run: TraceRun,
    enrichment: Mapping[IPAddress, HopEnrichment],
    thresholds: Thresholds = DEFAULT_THRESHOLDS,
) -> AnalysisResult: ...
```

- Pure function: no I/O, no clock, no randomness, no logging side effects. The same input always
  gives the same output (Hypothesis property P4, see [testing.md](../testing.md#hypothesis-properties)).
- `enrichment` may be empty. The classifier must still work and must name locations by hop
  number only ("between hops 6 and 7").
- Modules: `analyze/stats.py` (math), `analyze/hops.py` (per-hop aggregation), `analyze/path.py`
  (path stability), `analyze/attribution.py` (effective ASN, IXP confirmation, geo plausibility),
  `analyze/classifier.py` (decision logic), `analyze/wording.py` (sentences), `analyze/thresholds.py`.
- Out of scope: multi-vantage analysis, BGP correlation, MDA/multipath statistics, reverse traces.

## 2. Thresholds

Every number the classifier uses is a field of the frozen dataclass `Thresholds` in
`analyze/thresholds.py`. Users can override them in the config file (`[thresholds]`, see
[architecture.md](../architecture.md#configuration)). The JSON report lists any non-default values.
Agents: do not change a default without asking. <!-- Why: defaults are part of the product's honesty contract; snapshot tests pin them. -->

"Source" says where the value comes from. **Sourced** means a value or formula from the research.
**Suggested** means a design choice by the researcher or architect. Suggested values are
starting points that the M6 lab runs may revise.

| Name | Default | Unit | Source | Meaning |
|---|---|---|---|---|
| `WILSON_Z` | 1.96 | z | Sourced: 95% Wilson interval, [NIST 7.2.4.1](https://itl.nist.gov/div898/handbook/prc/section2/prc241.htm) | z used for every interval |
| `MIN_ROUNDS_ABSOLUTE` | 20 | rounds | Suggested | Fewer completed rounds means `INSUFFICIENT_DATA`, always |
| `ROUNDS_FOR_HIGH_CONFIDENCE` | 100 | rounds | Suggested from binomial math in loss_attribution Q4 ("≥100 cycles to call loss ≥2%") | Destination probes needed for `high` |
| `ROUNDS_FOR_MEDIUM_CONFIDENCE` | 50 | rounds | Suggested | Below this, confidence is capped at `low` |
| `LOSS_PRACTICAL` | 0.02 | fraction | Suggested (notes: 100 rounds resolve about 2%) | Observed loss at or above this can be "lossy"; below it can be "clean" |
| `LOSS_MIN_LOWER_BOUND` | 0.01 | fraction | Suggested | Wilson lower bound must reach this for "lossy" |
| `LOSS_CLEAN_MAX_UPPER` | 0.12 | fraction | Suggested (0/29 is the smallest all-clean run that passes, which matches quick mode = 30) | Wilson upper bound must be below this for "clean" |
| `CLEAN_HIGH_MAX_UPPER` | 0.04 | fraction | Suggested (0/100 gives 3.70%) | Clean verdicts with an upper bound below this get `high` |
| `CLEAN_MEDIUM_MAX_UPPER` | 0.08 | fraction | Suggested | Clean verdicts with an upper bound below this get `medium` (otherwise `low`) |
| `SILENT_HOP_MAX_REPLY_RATE` | 0.10 | fraction | Suggested | A hop with a reply rate below this is treated as silent (neutral) |
| `RUNS_MIN_EACH` | 5 | count | Suggested | The runs test needs at least this many replies AND this many losses |
| `RUNS_Z` | 2.0 | z | Suggested | `z ≥ RUNS_Z` means alternation (rate-limit pattern); `z ≤ −RUNS_Z` means bursty |
| `PATH_CHANGE_MIN_SHARE` | 0.10 | fraction | Suggested | Non-primary responders' share of a TTL's replies that counts as a path change |
| `PATH_CHANGE_MIN_REPLIES` | 3 | count | Suggested | Minimum non-primary replies that count as a path change |
| `LATENCY_MIN_SAMPLES` | 20 | replies | Suggested | A hop needs this many RTT samples to take part in latency analysis |
| `LATENCY_STEP_MIN_MS` | 10.0 | ms | Suggested. Fontugne et al. discard changes under 1 ms in a different, multi-vantage longitudinal setting ([arXiv 1605.04784](https://arxiv.org/pdf/1605.04784)). A single vantage point plus slow-path noise justifies a larger floor | Smallest median step considered |
| `LATENCY_PERSIST_FRACTION` | 0.8 | fraction | Suggested in loss_attribution Q3 ("median_j ≥ median_{k−1} + 0.8·Step_k") | Share of the step that must persist at every later hop |
| `PROPAGATION_MIN_SHARE` | 0.8 | fraction | Suggested | `step_min ≥ this × step_median` makes a step propagation-like |
| `FIBER_KM_PER_MS_RTT` | 100.0 | km/ms | Sourced: about 200 km/ms one way in fiber ([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf)) | Physics bound |
| `ROUTE_INFLATION` | 2.0 | factor | Suggested (the commonly quoted 1.5–2× was **not sourced**; see research gaps) | Allowed path stretch over great-circle distance |
| `LATENCY_EXCESS_MIN_MS` | 25.0 | ms | Suggested | Excess over the physics allowance that counts as a detour |
| `MPLS_RUN_MIN_HOPS` | 3 | hops | Suggested in loss_attribution Q2 ("run of ≥3 consecutive hops") | Length of a run of repeating latency values |
| `MPLS_FLAT_TOLERANCE_MS` | 2.0 | ms | Suggested | Hops count as "near-identical" when within max(this, `MPLS_FLAT_TOLERANCE_FRAC` × value) |
| `MPLS_FLAT_TOLERANCE_FRAC` | 0.05 | fraction | Suggested | See above |
| `JITTER_HIGH_MS` | 30.0 | ms | Suggested | Destination jitter that produces an informational finding |

Probing defaults (rounds 100, quick 30, interval 1 s) are not classifier thresholds. They live in
[cli.md](cli.md) and [ADR-0003](../decisions/0003-measurement-strategy.md).

## 3. Statistics (`analyze/stats.py`)

**Wilson score interval** for `x` losses out of `n` probes, with p̂ = x/n and z = `WILSON_Z`
([NIST](https://itl.nist.gov/div898/handbook/prc/section2/prc241.htm)):

```
center = p̂ + z²/(2n)
margin = z · sqrt( p̂(1−p̂)/n + z²/(4n²) )
lo, hi = (center − margin)/(1 + z²/n), (center + margin)/(1 + z²/n)     # clamp to [0, 1]
```

Reference values (computed with the formula above; they match the table in the research notes):

| lost/sent | observed | Wilson 95% CI |
|---|---|---|
| 0/10 | 0% | 0 – 27.8% |
| 1/10 | 10% | 1.8 – 40.4% |
| 2/10 | 20% | 5.7 – 51.0% |
| 0/30 | 0% | 0 – 11.4% |
| 1/30 | 3.3% | 0.59 – 16.7% |
| 0/100 | 0% | 0 – 3.7% |
| 1/100 | 1% | 0.18 – 5.45% |
| 2/100 | 2% | 0.55 – 7.00% |
| 3/100 | 3% | 1.03 – 8.45% |
| 5/100 | 5% | 2.15 – 11.2% |
| 6/100 | 6% | 2.78 – 12.5% |
| 0/300 | 0% | 0 – 1.26% |
| 3/300 | 1% | 0.34 – 2.90% |
| 15/300 | 5% | 3.05 – 8.08% |
| 0/1000 | 0% | 0 – 0.38% |
| 10/1000 | 1% | 0.54 – 1.83% |

Unit tests must reproduce every row to 2 decimal places in percent.

**Difference of two proportions (Newcombe hybrid score interval).** For p₁ = x₁/n₁ and p₂ = x₂/n₂
with Wilson intervals (l₁,u₁) and (l₂,u₂):

```
d  = p₁ − p₂
lo = d − sqrt((p₁ − l₁)² + (u₂ − p₂)²)
hi = d + sqrt((u₁ − p₁)² + (p₂ − l₂)²)
```

`significantly_greater(a, b)` is true if and only if `lo > 0` for d = p_a − p_b. This is the
"Newcombe/Wilson difference interval" option named in loss_attribution Q4. Fisher's exact test
is the other option there. We use Newcombe because it is closed-form and reuses Wilson.
Reference: 6/100 vs 0/100 gives d = 6%, CI [1.09%, 12.5%], so it is significant. 6/100 vs 5/100
gives CI [−5.96%, 8.07%], so it is not significant.

**Median and its CI.** Median = `statistics.median`. The 95% CI uses order statistics with Wilson
bounds at p = 0.5, as in Fontugne et al. ([arXiv 1605.04784](https://arxiv.org/pdf/1605.04784)).
Sort the samples into x₍₁₎…x₍ₙ₎ and compute (L, U) = Wilson(n/2 successes, n). The CI is
[x₍max(1, ⌊nL⌋)₎, x₍min(n, ⌈nU⌉)₎]. For n = 100 that is [x₍₄₀₎, x₍₆₀₎]; for n = 10 it is [x₍₂₎, x₍₈₎].

**p90** uses the nearest-rank rule: x₍⌈0.9n⌉₎. **Jitter** is the mean of |rttᵢ − rttᵢ₋₁| over consecutive
received samples in round order. This is our own definition, similar to mtr's jitter column. It is
informational only.

**Runs test (Wald–Wolfowitz)** on a hop's per-round reply sequence (1 = reply, 0 = no reply), with
n₁ replies, n₀ losses, n = n₁ + n₀ and R runs:

```
μ = 1 + 2·n₁·n₀/n
σ² = 2·n₁·n₀·(2·n₁·n₀ − n) / (n²·(n − 1))
z = (R − μ)/σ          # only if n₁ ≥ RUNS_MIN_EACH and n₀ ≥ RUNS_MIN_EACH, else None
```

`z ≥ RUNS_Z` means more alternation than random loss would produce. That fits token-bucket ICMP rate
limiting, which Guo & Heidemann describe as frequent alternation
([PAM 2018](https://www3.isi.edu/~johnh/PAPERS/Guo18a.pdf)). `z ≤ −RUNS_Z` means the losses come in
bursts. The runs test is our simple stand-in for their detector, not their method. A pattern is
evidence for rate limiting; its absence is not evidence against it.

## 4. Per-hop aggregation (`analyze/hops.py`)

Inputs come from `TraceRun.rounds` (only rounds with `status == OK`).

1. **Destination TTL `d*`.** For each round, find the smallest TTL whose reply came from
   `spec.target_ip` with a destination reply kind (`ReplyKind.DESTINATION`). `d*` is the most
   common value across rounds; ties go to the smaller TTL. If no round reached the destination,
   `d* = None`.
2. **Rows.** Rows are built for TTL `spec.first_ttl … d*`, or up to the highest TTL probed if
   `d* is None`. Replies at TTLs above `d*` are ignored. They come from rounds where the probe at
   `d*` was lost and the trace continued.
3. **Per row:** `sent` is the number of rounds where a probe was sent at that TTL. Whether this
   can be derived from scamper JSON is UNVERIFIED and must be confirmed in M0
   ([probing.md](probing.md#m0-spike-checklist)). `received` is the number of probes with a reply.
   `loss = 1 − received/sent`, plus `loss_ci`. RTT stats (min, median, median CI, p90, jitter) are
   computed over received probes. The row also keeps `reply_sequence` (per-round booleans),
   `runs_z`, `responders` (IP → reply count), `primary_responder` (most frequent; ties go to the
   lowest IP) and `mpls_labels_seen` (true if any reply carried RFC 4950 labels).
4. **Row kind:**
   - `DESTINATION`: TTL == `d*`.
   - `SILENT`: received == 0.
   - `MOSTLY_SILENT`: 0 < received/sent < `SILENT_HOP_MAX_REPLY_RATE`.
   - `UNREACHABLE_REPLY`: the majority reply is an ICMP destination-unreachable from a
     non-target address.
   - `ROUTER`: anything else.
   `SILENT` and `MOSTLY_SILENT` rows are **neutral**: comparisons skip them, and they never break
   persistence ([APNIC/Lavin](https://blog.apnic.net/2022/03/28/how-to-properly-interpret-a-traceroute-or-mtr/)).
   "Responding hops" below means `ROUTER` and `DESTINATION` rows.

**Loss state** of any row with sent ≥ 1:

```
LOSSY     if p̂ ≥ LOSS_PRACTICAL and wilson_lo ≥ LOSS_MIN_LOWER_BOUND
CLEAN     if p̂ <  LOSS_PRACTICAL and wilson_hi <  LOSS_CLEAN_MAX_UPPER
AMBIGUOUS otherwise
```

Examples: 3/100 is LOSSY, 2/100 is AMBIGUOUS, 1/100 is CLEAN, 0/30 is CLEAN (upper 11.35%),
1/30 is AMBIGUOUS and 0/20 is AMBIGUOUS.

## 5. Decision procedure (`analyze/classifier.py`)

Pseudocode. Names match the code. Every early `return` also attaches the findings collected so far.

```python
def classify(run, enrichment, t):
    rounds = [r for r in run.rounds if r.status is RoundStatus.OK]
    if len(rounds) < t.MIN_ROUNDS_ABSOLUTE:
        return result(INSUFFICIENT_DATA, finding=TOO_FEW_ROUNDS)            # §5.1

    hops = aggregate(rounds, run.spec, t)                                    # §4
    hop_ctx = attribute(hops, enrichment, t)        # effective ASN, IXP, geo plausibility (§6)
    annotate_neutral(hops)                          # finding SILENT_HOP per SILENT / MOSTLY_SILENT row

    change = detect_path_change(rounds, hops, t)                             # §5.2
    if change:
        return result(PATH_UNSTABLE, finding=PATH_CHANGE)

    dst = hops.destination
    if dst is None:                                                          # §5.3
        return result(DESTINATION_UNREACHABLE, finding=DESTINATION_SILENT
                      or DESTINATION_UNREACHABLE_ICMP)

    ref = dst
    annotate_icmp_deprioritized(hops, ref, t)                                # §5.4
    annotate_slow_path_latency(hops, t)                                      # §5.6
    mpls_runs = detect_mpls(hops, t)                                         # §5.7

    loss_state = dst.loss_state
    latency = find_latency_onset(hops, hop_ctx, mpls_runs, t)                # §5.6

    if loss_state is LOSSY:
        onset = find_loss_onset(hops, t)                                     # §5.5
        return result(LOSS_STARTS_AT, location=locate(onset, hop_ctx),
                      secondary=latency, confidence=loss_confidence(...))    # §7
    if latency is not None:
        return result(LATENCY_STARTS_AT, location=locate(latency, hop_ctx),
                      confidence=latency_confidence(...),
                      extra_caveat=LOSS_INCONCLUSIVE if loss_state is AMBIGUOUS else None)
    if loss_state is AMBIGUOUS:
        return result(INSUFFICIENT_DATA, finding=LOSS_INCONCLUSIVE)
    # loss_state is CLEAN and there is no persistent latency problem
    if any(h.flags & {ICMP_DEPRIORITIZED, SLOW_PATH_ARTIFACT} for h in hops):
        return result(ICMP_DEPRIORITIZATION_ONLY, confidence=clean_confidence(dst, t))
    return result(NO_PROBLEM, confidence=clean_confidence(dst, t))
```

Precedence is therefore INSUFFICIENT_DATA (too few rounds), then PATH_UNSTABLE, then
DESTINATION_UNREACHABLE, then LOSS_STARTS_AT, then LATENCY_STARTS_AT, then INSUFFICIENT_DATA (loss
ambiguous), then ICMP_DEPRIORITIZATION_ONLY, then NO_PROBLEM.

### 5.1 Too few rounds
If fewer than `MIN_ROUNDS_ABSOLUTE` rounds completed, return `INSUFFICIENT_DATA` whatever the
data shows. 10 rounds (mtr's report default) cannot rule out 27.8% loss (Wilson, 0/10).

### 5.2 Path change (`analyze/path.py`)
A TTL ≤ `d*` (or any TTL when `d*` is None) is **unstable** if its non-primary responders sent at
least `PATH_CHANGE_MIN_REPLIES` replies **and** at least `PATH_CHANGE_MIN_SHARE` of the row's
replies. The destination TTL is unstable if at least `PATH_CHANGE_MIN_REPLIES` rounds (and at least
`PATH_CHANGE_MIN_SHARE` of rounds) reached the destination at a TTL **below** `d*`. Any unstable TTL
gives `PATH_UNSTABLE`.

The finding describes the pattern. Run the runs test on the per-round indicator "primary
responder or not". If z ≤ −`RUNS_Z`, report "route change at about round N", where N is the first
round of the longest non-primary block. Otherwise report "interleaved responders (per-packet load
balancing, or the flow tuple was not held constant)". With Paris probing and fixed ports, the
interleaved case should not happen. If it does, that is a backend bug worth investigating
([Augustin et al.](https://arxiv.org/pdf/0904.2733)).
Do not merge the stats silently. Per-segment analysis is a stretch goal ([roadmap](../roadmap.md#stretch-goals)).

### 5.3 Destination never answered
If `d*` is None: if any responding row is `UNREACHABLE_REPLY`, use finding
`DESTINATION_UNREACHABLE_ICMP` and name that hop and its ICMP type/code. Otherwise use
`DESTINATION_SILENT` and name the last responding hop. The verdict is `DESTINATION_UNREACHABLE` and
confidence is `null`. End-to-end loss cannot be assessed. The wording must say that a firewall
dropping this probe type and a real outage look the same here, and suggest another protocol
([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf);
[APNIC/Lavin](https://blog.apnic.net/2022/03/28/how-to-properly-interpret-a-traceroute-or-mtr/)).
`ICMP_DEPRIORITIZED` annotations (§5.4) are still computed for the evidence, with the last
responding row of the trace as the reference instead of the destination (rows before it only).

### 5.4 ICMP deprioritization (cosmetic loss)
A responding `ROUTER` row j < `d*` gets flag `ICMP_DEPRIORITIZED` when:

```
p̂_j ≥ LOSS_PRACTICAL  and  significantly_greater(row_j, ref)      # ref = destination row
```

Also set `RATE_LIMIT_PATTERN` when `runs_z ≥ RUNS_Z`. That raises nothing by itself but goes into
the evidence. Rationale: loss in a hop's own replies says nothing about traffic forwarded through
it ([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf);
[Cloudflare](https://www.cloudflare.com/learning/network-layer/what-is-mtr/)). The flag is set
even when the destination is itself lossy. In that case it marks the **excess** loss at hop j.

### 5.5 Loss onset (only when the destination is LOSSY)
Row j is **clearly cleaner than the destination** if `significantly_greater(dst, row_j)`.
Row j is **confirmed degraded** if `p̂_j ≥ LOSS_PRACTICAL` and j is not `ICMP_DEPRIORITIZED`.
A deprioritized row has more loss than the destination, so its own replies cannot tell whether
forwarding loss starts before or after it.

```python
responding = [r for r in hops if r.kind in (ROUTER, DESTINATION)]     # ordered by TTL, ends at dst
last_clean = None
for r in reversed(responding[:-1]):            # walk back from the hop before dst
    if clearly_cleaner_than_dst(r):
        last_clean = r
        break
suffix = [r for r in responding if last_clean is None or r.ttl > last_clean.ttl]
onset = next(r for r in suffix if confirmed_degraded(r))       # exists: dst itself qualifies
```

- `last_clean` is the last hop with positive evidence of being clean. `onset` is the first
  degraded hop. The location is the span `last_clean → onset`; usually they are adjacent.
- Responding rows in the suffix before `onset` (low observed loss that is still not clearly cleaner,
  or deprioritized rows) add caveat `ONSET_RANGE`: "hops {a}–{b} show too little loss to place the
  start precisely; it is somewhere between hops {c} and {k}".
- Neutral rows between `last_clean` and `onset` add caveat `SILENT_GAP`.
- Rate limiters before `last_clean` never move the onset (lab scenario 4). Sampling noise in rows
  before the true onset cannot pull it upstream either, because a row with no losses is never
  confirmed degraded (property P2).
- If `last_clean is None`, no hop is clearly loss-free. The location kind is `LOCAL` when `onset` is
  private, CGNAT or TTL ≤ 2, and `PATH_START` otherwise.
- If `onset is dst`, the location kind is `DESTINATION`. Loss that appears only at the destination
  may be the host, its network, or the return path
  ([Cloudflare](https://www.cloudflare.com/learning/network-layer/what-is-mtr/)). A host that
  rate-limits its own replies looks the same. If `dst.runs_z ≥ RUNS_Z`, add caveat
  `DESTINATION_HOST_RATE_LIMIT`.
- If `dst.runs_z ≤ −RUNS_Z`, add finding `BURSTY_LOSS` and caveat `BURSTY_CI_OPTIMISTIC`. Bursty
  loss breaks the binomial independence assumption, so the intervals are too narrow (loss_attribution Q4).
- Not used in v1: cycle-aligned paired analysis and multiple-comparison corrections. The
  "every downstream hop must agree" rule already guards against single-hop false positives.

### 5.6 Latency
Only rows with `received ≥ LATENCY_MIN_SAMPLES` take part ("eligible"). If the destination is not
eligible, there is no latency verdict. Instead add finding `LATENCY_UNCONFIRMABLE` when a step
exists upstream.

**Slow-path artifact.** Eligible router row j gets `SLOW_PATH_ARTIFACT` when some later eligible
row m (including the destination) has
`median_j − median_m ≥ LATENCY_STEP_MIN_MS` and `median_ci_j.lo > median_ci_m.hi`. In words: this
hop is slower than a hop behind it, so the extra time is spent generating ICMP and not forwarding
([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf)).
Max, mean and stdev never drive a verdict.

**Step candidates.** Walk eligible rows in TTL order. For row k, let p be the previous eligible row
that is not `SLOW_PATH_ARTIFACT` (an inflated predecessor would hide a real step):

```
step_med = median_k − median_p
step_min = min_k − min_p
significant = step_med ≥ LATENCY_STEP_MIN_MS and median_ci_k.lo > median_ci_p.hi
persistent  = all(median_j ≥ median_p + LATENCY_PERSIST_FRACTION·step_med
                  for every eligible row j after k, destination included)
```

Only the **first** k that is significant and persistent is considered. Classify it:

- **Queueing-like**: `step_min < PROPAGATION_MIN_SHARE · step_med`. The floor stayed roughly flat
  while typical delay rose. Result: `LATENCY_STARTS_AT` at link (p → k).
- **Propagation-like**: otherwise (min rose about as much as the median). Use the physics check
  from §6 when both p and k have geolocation with confidence ≥ medium:
  `allowance = ROUTE_INFLATION · distance_km(p, k) / FIBER_KM_PER_MS_RTT`. If
  `step_min − allowance ≥ LATENCY_EXCESS_MIN_MS`, the result is `LATENCY_STARTS_AT` with finding
  `DETOUR` (more delay than the distance explains). Otherwise there is no latency problem. Without
  usable geolocation there is also no latency problem. In both no-problem cases, add the
  informational finding `PROPAGATION_STEP` ("+68 ms between hops 5 and 6 that persists; the
  minimum RTT rose as much as the median, which points to distance or an MPLS tunnel, not
  congestion"). <!-- Why: an ocean crossing is not a fault; without location evidence we must not call it one (ADR-0007). -->

A persistent step whose row k lies inside an MPLS run (§5.7) is attributed to the run (ingress →
egress), never to an interior hop.

Every `LATENCY_STARTS_AT` result gets caveat `RETURN_PATH`. RTT includes an unseen return path,
and only a reverse trace can rule it out
([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf);
[Fontugne et al.](https://arxiv.org/pdf/1605.04784)).
Destination jitter ≥ `JITTER_HIGH_MS` adds the informational finding `HIGH_JITTER`. Jitter at
intermediate hops is ignored.

### 5.7 MPLS
`MPLS_SUSPECTED` covers a run of rows a..b when either condition holds:

- (a) any row in the run has `mpls_labels_seen` (RFC 4950 extensions,
  [RFC 4950](https://www.rfc-editor.org/rfc/rfc4950.txt); scamper field names are UNVERIFIED, M0); or
- (b) at least `MPLS_RUN_MIN_HOPS` consecutive eligible `ROUTER` rows (the destination is never
  part of a run) have min RTTs within `max(MPLS_FLAT_TOLERANCE_MS, MPLS_FLAT_TOLERANCE_FRAC × m₀)`
  of the run's first row's min RTT m₀, **and** m₀ exceeds the previous eligible row's min RTT by at
  least `LATENCY_STEP_MIN_MS`. This is the "repeating latency" signature
  ([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf)).
  Without the step-up requirement, any flat stretch near the source would match.

Effects: latency onset inside a run is attributed to "the MPLS segment between hops p and b", where
p is the eligible row before the run and b is the run's last row.
Confidence is lowered (§7). Caveat `MPLS_HIDDEN_HOPS` is added when the run touches the onset,
because LSP hops may be hidden ([RFC 3443](https://www.rfc-editor.org/rfc/rfc3443.txt);
[Donnet et al.](https://www.caida.org/catalog/papers/2012_revealing_mpls_tunnels/revealing_mpls_tunnels.pdf)).

## 6. Attribution (`analyze/attribution.py`)

This is pure logic over `HopStats` and the per-IP `HopEnrichment` facts from `enrich/`.

**Effective ASN** of each row's primary responder, in this order:

1. IXP member exact-IP match: the member ASN (`asn_basis = IXP_MEMBER`). This overrides BGP origin
   ([traIXroute](https://labs.ripe.net/author/george_nomikos/detecting-ixps-in-traceroute-paths-using-traixroute/)).
2. Public address with origin ASN(s): the first origin ASN, with all of them kept in `origin_asns`
   (`ORIGIN`).
3. Private or CGNAT address between two public rows with the same effective ASN X: X
   (`INFERRED_FROM_NEIGHBORS`). A CGNAT address before the first public row inherits that row's
   ASN (`INFERRED_FROM_NEIGHBORS`). Private rows before the first public row are `LOCAL`.
4. Otherwise None (`NONE`).

**IXP crossing confirmation.** An IXP-fabric row counts as a confirmed crossing only if the
previous responding public row's effective ASN is a member of the same IX, **or** the next row's
effective ASN equals the member ASN. Both true gives `high`, one gives `medium`. A confirmed crossing
adds the informational finding `IXP_CROSSING`. Otherwise the hop is labeled "IXP address seen (crossing
not confirmed)". A peering-LAN address alone is not conclusive
([traIXroute](https://labs.ripe.net/author/george_nomikos/detecting-ixps-in-traceroute-paths-using-traixroute/)).

**Location of an onset.** c = last clean hop (loss) or p (latency), k = onset. The first matching
row wins:

| Kind | Condition | Wording fragment |
|---|---|---|
| `MPLS_SEGMENT` | latency onset inside an MPLS run (§5.7) | `in an MPLS segment, between hops {p} and {b}` (+ ` inside AS{X} ({name})` if p..b share one effective ASN) |
| `DESTINATION` | onset is the destination row | `only at the destination (hop {k}{, AS…})` |
| `LOCAL` | no clean hop before onset; onset private, CGNAT or TTL ≤ 2 | `in your local network or access link (by hop {k})` |
| `PATH_START` | no clean hop before onset (otherwise) | `somewhere before hop {k}; no earlier hop is clearly loss-free` |
| `UNKNOWN_AS` | the effective ASN of c or k is None | `between hops {c} and {k}` |
| `AS_BOUNDARY` | responding rows c..k carry ≥ 2 distinct known effective ASNs | `between AS{X} ({nx}) and AS{Y} ({ny}), between hops {c} and {k}` (X = AS of c, Y = AS of k; + ` at {IXP}` if a confirmed crossing lies in c..k; further ASNs go in the evidence) |
| `SAME_AS` | otherwise (one known ASN X across c..k) | `inside AS{X} ({name}), between hops {c} and {k}` |

Never say "in AS Y" for a boundary. Interconnect addresses often come from one side's space, so
the first hop with AS Y's address may still be AS X's router
([bdrmap](https://users.caida.org/~mjl/pubs/bdrmap.pdf)). `AS_BOUNDARY` always adds caveat `AS_BOUNDARY_SIDE`.

**Geo plausibility.** For consecutive geolocated rows a and b:
if `great_circle_km(a, b) > FIBER_KM_PER_MS_RTT × (min_a + min_b)`, at least one location is
physically impossible. Flag `GEO_IMPLAUSIBLE` on the lower-confidence one (both if they are equal)
and do not use it in the propagation check. This only needs RTTs: d(a, b) ≤ d(v, a) + d(v, b) ≤
100·(RTT_a + RTT_b). Rationale: databases geolocate routers poorly
([Gharaibeh et al. via APNIC](https://blog.apnic.net/2017/11/03/trust-geolocation-databases-geolocate-routers/)).

## 7. Confidence

`confidence` is `high`, `medium`, `low`, or `null` for the inconclusive verdicts
(`INSUFFICIENT_DATA`, `PATH_UNSTABLE`, `DESTINATION_UNREACHABLE`). Start at the base level and apply
**caps**; the lowest cap wins. `n_dst` = probes sent at the destination row.

**NO_PROBLEM, ICMP_DEPRIORITIZATION_ONLY** (base from the destination's Wilson upper bound):
`hi < CLEAN_HIGH_MAX_UPPER` gives high, `hi < CLEAN_MEDIUM_MAX_UPPER` gives medium, otherwise low.
So 0/100 is high (3.70%), 1/100 is medium (5.45%), 0/30 is low (11.35%).

**LOSS_STARTS_AT** (base high):

| Cap | Condition |
|---|---|
| medium | `n_dst < ROUNDS_FOR_HIGH_CONFIDENCE` |
| low | `n_dst < ROUNDS_FOR_MEDIUM_CONFIDENCE` |
| medium | location kind `AS_BOUNDARY`, `UNKNOWN_AS` or `PATH_START` |
| medium | `SILENT_GAP` or `ONSET_RANGE` caveat |
| medium | onset is the destination row |
| low | onset is the destination row and `DESTINATION_HOST_RATE_LIMIT` |
| medium | `BURSTY_CI_OPTIMISTIC` |
| medium | MPLS run touches onset |
| medium | Newcombe lower bound of (p_dst − p_last_clean) < `LOSS_MIN_LOWER_BOUND` |

**LATENCY_STARTS_AT** (base medium, because the return path can never be ruled out from one side):
capped at low if `n_dst < ROUNDS_FOR_HIGH_CONFIDENCE`, if there is a `SILENT_GAP` (a neutral row
between the previous eligible row p and k), if an MPLS run touches the onset, or if the result is
`DETOUR` (geolocation is never better than medium).

## 8. Wording (`analyze/wording.py`)

There is one summary sentence per verdict, followed by evidence lines, caveats and hints.
Percentages use 1 decimal place, CIs use 1 decimal place, and ms values use 1 decimal place.
`{as}` renders as `AS3356 (Lumen)`, or `AS3356` when there is no name. `{protocol}` renders as
`UDP`, `ICMP` or `TCP port {dport}`.

| Verdict | Template |
|---|---|
| `NO_PROBLEM` | `No real problem: {lost}/{sent} probes to the destination were lost ({loss}%, 95% CI {lo}–{hi}%) and no latency problem was found.` |
| `ICMP_DEPRIORITIZATION_ONLY` | `No real problem; hop {k} just deprioritizes ICMP.` With several hops: `No real problem; hops {k1} and {k2} just deprioritize ICMP.` |
| `LOSS_STARTS_AT` | `Loss starts {location}.` e.g. `Loss starts inside AS3356 (Lumen), between hops 6 and 7.` |
| `LATENCY_STARTS_AT` | `Latency problem starts {location}.` e.g. `Latency problem starts inside AS3356 (Lumen), between hops 6 and 7.` |
| `DESTINATION_UNREACHABLE` | `Inconclusive: the destination never answered {protocol} probes; the last hop that answered was hop {k} in {as}.` (drop ` in {as}` when unknown) |
| `PATH_UNSTABLE` | `Inconclusive: the path changed during the run (hop {t}: {ip_a} in {n_a} rounds, {ip_b} in {n_b} rounds).` |
| `INSUFFICIENT_DATA` (rounds) | `Not enough data: only {n} rounds completed; at least {MIN_ROUNDS_ABSOLUTE} are needed.` |
| `INSUFFICIENT_DATA` (ambiguous) | `Not enough data: {lost}/{sent} probes to the destination were lost, so true loss is between {lo}% and {hi}%. Run with --rounds {suggest}.` with `suggest = max(300, 3 × rounds)` |

Evidence lines (always shown for onset verdicts):
`Destination loss {loss}% ({lost}/{sent}, 95% CI {lo}–{hi}%); hop {c} {loss_c}% ({lost_c}/{sent_c}); every hop from {k} to {d} shows comparable loss.`
For latency: `Median RTT rises {step} ms at hop {k} (min RTT +{step_min} ms) and stays ≥ {floor} ms through the destination.`

Caveat texts (stable codes, used in JSON):

| Code | Text |
|---|---|
| `RETURN_PATH` | `RTT includes the unseen return path; the delay may be on the way back. Only a trace from the far end can tell.` |
| `AS_BOUNDARY_SIDE` | `The problem is at an interconnection; either network, or the return path, may be responsible.` |
| `SILENT_GAP` | `Hops {a}–{b} did not reply; the problem may start anywhere between hops {c} and {k}.` |
| `ONSET_RANGE` | `Hops {a}–{b} show too little loss to place the start precisely; it is somewhere between hops {c} and {k}.` |
| `MPLS_HIDDEN_HOPS` | `Hops {a}–{b} look like an MPLS tunnel; hidden routers may sit inside it and RTTs reflect the tunnel exit.` |
| `DESTINATION_HOST_RATE_LIMIT` | `The destination answers in an on/off pattern typical of ICMP rate limiting; the "loss" may be the host limiting replies.` |
| `BURSTY_CI_OPTIMISTIC` | `Losses came in bursts; the confidence intervals assume independent losses and are too narrow.` |
| `QUICK_MODE` | `Quick mode ({n} rounds) can only rule out loss above about {hi}%; smaller loss may go unnoticed.` ({hi} is the destination's Wilson upper bound, 11.4% for 0/30) |
| `LOSS_INCONCLUSIVE` | `End-to-end loss is inconclusive at this sample size.` |
| `ENRICHMENT_DEGRADED` | `Some network names are missing ({reason}); run 'spa data update'.` |

Hints ("what would raise confidence"), chosen by condition: more rounds (`--rounds 300`); another
probe type (`--protocol tcp --dport 443` or `--protocol icmp`); a trace from the far end; a
different source address (Steenbergen's technique).

## 9. Worked examples (acceptance tests for M2)

Each example becomes a test in `tests/unit/analyze/test_worked_examples.py`. The test builds the
`TraceRun` with the synthetic builder ([testing.md](../testing.md#synthetic-builder)) and asserts
verdict, confidence, location and flags. "—" means silent. RTTs are in ms. Unless stated
otherwise: 100 rounds, UDP-Paris, all hops public and enriched as shown.

### E1: Clean path, single stray loss → NO_PROBLEM (high), exit 0

| hop | AS | lost/sent | min | median |
|---|---|---|---|---|
| 1 | private | 0/100 | 0.5 | 0.7 |
| 2–7 | various | 0/100 (hop 3: 1/100) | 3–20 | 4–22 |
| 8 (dst) | AS13335 | 0/100 | 21.0 | 22.4 |

Hop 3's 1% is below `LOSS_PRACTICAL`, so it gets no flag. The destination is CLEAN with upper
bound 3.70%, which gives high confidence.
Summary: `No real problem: 0/100 probes to the destination were lost (0.0%, 95% CI 0.0–3.7%) and no latency problem was found.`

### E2: Rate-limited middle hop → ICMP_DEPRIORITIZATION_ONLY (high), exit 0

| hop | AS | lost/sent | pattern |
|---|---|---|---|
| 1–3 | AS7922 | 0/100 | |
| 4 | AS3356 | 30/100 | lost on odd rounds during rounds 0–59 → runs z = +4.32 |
| 5–8 | AS3356 | 0/100 | |
| 9 (dst) | AS15169 | 0/100 | |

Hop 4 vs dst: Newcombe CI of the difference is [21.1%, 39.6%], so hop 4 is significantly greater
and gets `ICMP_DEPRIORITIZED`. It also gets `RATE_LIMIT_PATTERN`. The destination is CLEAN (upper 3.70%).
Summary: `No real problem; hop 4 just deprioritizes ICMP.`

### E3: Loss inside one AS → LOSS_STARTS_AT (high), exit 1

| hop | AS | lost/sent |
|---|---|---|
| 1–5 | AS7922, AS3356 | 0/100 |
| 6 | AS3356 | 0/100 |
| 7 | AS3356 | 7/100 |
| 8 | AS3356 | 6/100 |
| 9 | AS3356 | 9/100 |
| 10 (dst) | AS64500 | 6/100 |

The destination is LOSSY (6%, lower bound 2.78%). Walking back from hop 9: hops 9, 8 and 7 are not
clearly cleaner than 6/100. Hop 6 is (difference CI [1.09%, 12.5%]), so last_clean = 6. The first
confirmed degraded hop after it is hop 7 (7% ≥ 2%, not deprioritized), so onset = 7. Both are
AS3356, so the location is `SAME_AS`. The Newcombe lower bound is 1.09% ≥ 1%, so there is no cap.
Summary: `Loss starts inside AS3356 (Lumen), between hops 6 and 7.`

### E4: Loss at an AS boundary behind a rate-limited hop → LOSS_STARTS_AT (medium), exit 1

| hop | AS | lost/sent |
|---|---|---|
| 1–2 | AS7922 | 0/100 |
| 3 | AS7922 | 41/100 (alternating) |
| 4 | AS7922 | 0/100 |
| 5 | AS3356 | 5/100 |
| 6–8 | AS3356 | 5–8/100 |
| 9 (dst) | AS64500 | 6/100 |

Onset = 5 and last_clean = 4. Hop 3 gets `ICMP_DEPRIORITIZED` (41 vs 6, significant) and does not
affect the onset. The location is `AS_BOUNDARY`, which caps confidence at medium and adds `AS_BOUNDARY_SIDE`.
Summary: `Loss starts between AS7922 (Comcast) and AS3356 (Lumen), between hops 4 and 5.`

### E5: Queueing latency step → LATENCY_STARTS_AT (medium), exit 1

| hop | AS | min | median (95% CI) | lost/sent |
|---|---|---|---|---|
| 5 | AS3356 | 10.8 | 11.6 (11.3–11.9) | 0/100 |
| 6 | AS3356 | 11.0 | 12.0 (11.7–12.4) | 0/100 |
| 7 | AS3356 | 12.0 | 48.0 (44.1–52.3) | 0/100 |
| 8 | AS3356 | 13.0 | 50.0 (46.0–54.2) | 0/100 |
| 9 (dst) | AS64500 | 14.0 | 51.0 (47.2–55.0) | 0/100 |

At hop 7: step_med = 36.0 and the CIs are disjoint. Persistence floor = 12.0 + 0.8·36 = 40.8,
and 50.0 and 51.0 are both above it. step_min = 1.0 < 0.8·36, so the step is queueing-like.
Loss is CLEAN. Caveat `RETURN_PATH` applies.
Summary: `Latency problem starts inside AS3356 (Lumen), between hops 6 and 7.`

### E6: Cosmetic latency spike → ICMP_DEPRIORITIZATION_ONLY (high), exit 0

Hop 2 has median 10.0 (CI 9.8–10.3). Hop 3 has median 85.0 (CI 70.0–97.0) and min 9.0. Hop 4 has
median 11.0 (CI 10.8–11.3), and later hops are 11–14. Hop 3 gets `SLOW_PATH_ARTIFACT` (85 − 11 ≥ 10,
CIs disjoint). Hop 3's step (75 ms over hop 2) is not persistent: hop 4 at 11.0 is below the floor of
10 + 0.8·75 = 70. For hop 4, p skips hop 3 and is hop 2, so the step is 1 ms, which is not significant.
All rows are 0/100.
Summary: `No real problem; hop 3 just deprioritizes ICMP.`

### E7: Silent destination → DESTINATION_UNREACHABLE (null), exit 2

Hops 1–11 answer (hop 11 is AS15169). TTLs 12–16 never answer, and each round stops at the backend's
gap limit (scamper's gap-limit behavior is UNVERIFIED, M0).
Summary: `Inconclusive: the destination never answered UDP probes; the last hop that answered was hop 11 in AS15169 (Google).`
Hint: `--protocol icmp` or `--protocol tcp --dport 443`.

### E8: Too few rounds → INSUFFICIENT_DATA (null), exit 2

10 rounds and any data. Summary: `Not enough data: only 10 rounds completed; at least 20 are needed.`

### E9: Ambiguous loss → INSUFFICIENT_DATA (null), exit 2

100 rounds with dst at 2/100: p̂ = 2% (not clean), lower bound 0.55% < 1% (not lossy). No latency step.
Summary: `Not enough data: 2/100 probes to the destination were lost, so true loss is between 0.6% and 7.0%. Run with --rounds 300.`
The same rule makes quick mode with dst at 1/30 INSUFFICIENT_DATA, while quick mode with 0/30 gives
NO_PROBLEM (low) plus caveat `QUICK_MODE`.

### E10: Route change mid-run → PATH_UNSTABLE (null), exit 2

Hop 6 is 198.51.100.1 in rounds 0–54 and 198.51.100.77 in rounds 55–99. The non-primary share is
45% (≥ 10%) with 45 replies (≥ 3). The indicator runs z is far below −2, so the finding is "route
change at about round 55".
Summary: `Inconclusive: the path changed during the run (hop 6: 198.51.100.1 in 55 rounds, 198.51.100.77 in 45 rounds).`

### E11: Loss only at the destination → LOSS_STARTS_AT (medium), exit 1

All routers are 0/100 and dst is 8/100 with random pattern (runs z between −2 and 2). Onset = dst,
location `DESTINATION`, capped at medium. If dst losses alternate (z ≥ 2), the cap is low and caveat
`DESTINATION_HOST_RATE_LIMIT` is added.
Summary: `Loss starts only at the destination (hop 10, AS64500).`

### E12: Long-haul or MPLS step, no geo → NO_PROBLEM (high), exit 0

| hop | min | median |
|---|---|---|
| 5 | 2.0 | 2.4 |
| 6 | 70.1 | 71.0 |
| 7 | 70.4 | 71.3 |
| 8 | 70.0 | 70.9 |
| 9 | 70.5 | 71.4 |
| 10 (dst) | 72.0 | 72.6 |

The step at hop 6 is significant and persistent, but step_min (68.1) ≥ 0.8 × step_med (68.6), so it
is propagation-like. There is no geolocation, so the result is finding `PROPAGATION_STEP` rather than
a verdict. Hops 6–9 form an MPLS run (4 ≥ 3 with near-identical min RTTs), so `MPLS_SUSPECTED` is
added. All rows are 0/100.
Variant E12-geo: hop 5 has an rDNS hint `lon` and hop 6 has `nyc` (both medium). The distance is
about 5,540 km, so allowance = 2.0 × 55.4 = 110.8 ms. step_min 68.1 is within it, so there is still
no problem. If step_min were 150 ms, the excess of 39.2 ms ≥ 25 would give `LATENCY_STARTS_AT` with
finding `DETOUR`, location `MPLS_SEGMENT` (hop 6 starts the MPLS run) and confidence low (the MPLS
run touches the onset).
