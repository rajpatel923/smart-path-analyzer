# Spec: data model

Internal types are plain frozen dataclasses. Pydantic is used only at the JSON output boundary
(`report/json_models.py`). Rationale: [ADR-0001](../decisions/0001-language-and-tooling.md) and
[ADR-0008](../decisions/0008-output-and-exit-codes.md).
<!-- Why: dataclasses keep the pure analysis layer free of validation overhead and third-party types; Pydantic gives us a publishable JSON Schema. -->

Conventions:
- Every dataclass is `@dataclass(frozen=True, slots=True)`. Collections are `tuple[...]` or
  `frozenset[...]`, never `list` or `dict`. Mappings use `types.MappingProxyType` or a tuple of pairs.
- Units go in field names: `_ms` is milliseconds (float), `_s` is seconds (float). Loss is a
  fraction in [0, 1], never a percent. Timestamps are timezone-aware UTC `datetime`.
- `IPAddress = ipaddress.IPv4Address | ipaddress.IPv6Address`. v0.1 accepts only IPv4 targets,
  but every address field uses `IPAddress` so that IPv6 needs no model change
  ([roadmap stretch goals](../roadmap.md#stretch-goals)).
- TTL and hop number are the same thing. Hops are numbered from 1.

## Enums (`models.py`)

```python
class Protocol(StrEnum):      UDP = "udp"; ICMP = "icmp"; TCP = "tcp"      # all Paris variants
class ReplyKind(StrEnum):     TIME_EXCEEDED = "time_exceeded"; DESTINATION = "destination"
                              UNREACHABLE = "unreachable"; OTHER = "other"
class RoundStatus(StrEnum):   OK = "ok"; TIMEOUT = "timeout"; BACKEND_ERROR = "backend_error"
class RowKind(StrEnum):       ROUTER = "router"; DESTINATION = "destination"; SILENT = "silent"
                              MOSTLY_SILENT = "mostly_silent"; UNREACHABLE_REPLY = "unreachable_reply"
class LossState(StrEnum):     CLEAN = "clean"; LOSSY = "lossy"; AMBIGUOUS = "ambiguous"
class HopFlag(StrEnum):       ICMP_DEPRIORITIZED = "icmp_deprioritized"; RATE_LIMIT_PATTERN = "rate_limit_pattern"
                              SLOW_PATH_ARTIFACT = "slow_path_artifact"; MPLS_SUSPECTED = "mpls_suspected"
                              GEO_IMPLAUSIBLE = "geo_implausible"; PATH_UNSTABLE = "path_unstable"
class Verdict(StrEnum):       NO_PROBLEM = "NO_PROBLEM"
                              ICMP_DEPRIORITIZATION_ONLY = "ICMP_DEPRIORITIZATION_ONLY"
                              LOSS_STARTS_AT = "LOSS_STARTS_AT"
                              LATENCY_STARTS_AT = "LATENCY_STARTS_AT"
                              DESTINATION_UNREACHABLE = "DESTINATION_UNREACHABLE"
                              PATH_UNSTABLE = "PATH_UNSTABLE"
                              INSUFFICIENT_DATA = "INSUFFICIENT_DATA"
class Confidence(StrEnum):    HIGH = "high"; MEDIUM = "medium"; LOW = "low"
class LocationKind(StrEnum):  SAME_AS = "same_as"; AS_BOUNDARY = "as_boundary"; UNKNOWN_AS = "unknown_as"
                              LOCAL = "local"; PATH_START = "path_start"; DESTINATION = "destination"
                              MPLS_SEGMENT = "mpls_segment"
class AddressClass(StrEnum):  PUBLIC = "public"; PRIVATE = "private"; CGNAT = "cgnat"
                              LINK_LOCAL = "link_local"; LOOPBACK = "loopback"
                              DOCUMENTATION = "documentation"; BENCHMARK = "benchmark"
                              RESERVED = "reserved"; MULTICAST = "multicast"; SPECIAL_OTHER = "special_other"
class AsnBasis(StrEnum):      IXP_MEMBER = "ixp_member"; ORIGIN = "origin"
                              INFERRED_FROM_NEIGHBORS = "inferred_from_neighbors"; LOCAL = "local"; NONE = "none"
class GeoBasis(StrEnum):      RDNS_HINT = "rdns_hint"; DB_CITY = "db_city"; DB_COUNTRY = "db_country"
```

Verdict → exit code mapping is in [cli.md](cli.md#exit-codes). Finding and caveat codes are listed
in [classifier.md](classifier.md). They are `StrEnum`s `FindingCode` and `CaveatCode` whose values
equal their names. Finding codes: `TOO_FEW_ROUNDS`, `PATH_CHANGE`, `DESTINATION_SILENT`,
`DESTINATION_UNREACHABLE_ICMP`, `ICMP_DEPRIORITIZED`, `SLOW_PATH_ARTIFACT`, `SILENT_HOP`,
`BURSTY_LOSS`, `LOSS_INCONCLUSIVE`, `LATENCY_UNCONFIRMABLE`, `PROPAGATION_STEP`, `DETOUR`,
`MPLS_SUSPECTED`, `HIGH_JITTER`, `GEO_IMPLAUSIBLE`, `IXP_CROSSING`. Caveat codes: `RETURN_PATH`,
`AS_BOUNDARY_SIDE`, `SILENT_GAP`, `ONSET_RANGE`, `MPLS_HIDDEN_HOPS`, `DESTINATION_HOST_RATE_LIMIT`,
`BURSTY_CI_OPTIMISTIC`, `QUICK_MODE`, `LOSS_INCONCLUSIVE`, `ENRICHMENT_DEGRADED`.

## Measurement types (`models.py`, produced by `probe/`)

```python
@dataclass(frozen=True, slots=True)
class FlowId:
    protocol: Protocol
    sport: int | None            # UDP/TCP source port, fixed for the whole run
    dport: int | None            # UDP/TCP destination port, fixed for the whole run
    icmp_checksum: int | None    # ICMP-Paris: as reported by the backend (UNVERIFIED field, M0)

@dataclass(frozen=True, slots=True)
class ProbeSpec:
    target: str                  # as typed by the user (hostname or IP)
    target_ip: IPAddress         # resolved once, before round 0
    flow: FlowId
    first_ttl: int = 1
    max_ttl: int = 30
    rounds: int = 100
    interval_s: float = 1.0      # minimum start-to-start spacing of rounds
    wait_s: int = 2              # per-probe reply timeout passed to the backend
    gap_limit: int = 5           # consecutive silent TTLs before the backend stops a round
    round_timeout_s: float = 60.0
    quick: bool = False

@dataclass(frozen=True, slots=True)
class MplsLabel:
    label: int; exp: int; bottom_of_stack: bool; ttl: int     # RFC 4950 label stack entry

@dataclass(frozen=True, slots=True)
class ProbeReply:
    ttl: int                     # probe TTL
    responder: IPAddress
    rtt_ms: float
    kind: ReplyKind
    icmp_type: int | None        # None for TCP responses
    icmp_code: int | None
    reply_ttl: int | None        # IP TTL of the reply as received
    quoted_ttl: int | None       # TTL quoted inside the ICMP payload (q-TTL), MPLS hint
    mpls_labels: tuple[MplsLabel, ...] = ()

@dataclass(frozen=True, slots=True)
class Round:
    index: int                   # 0-based
    started_at: datetime
    duration_s: float
    status: RoundStatus
    probed_ttls: frozenset[int]  # TTLs a probe was SENT at in this round (UNVERIFIED derivation, M0)
    replies: tuple[ProbeReply, ...]   # at most one per TTL (we send one attempt per TTL)
    stop_reason: str | None      # backend's value, verbatim
    error: str | None = None     # set when status != OK

@dataclass(frozen=True, slots=True)
class TraceRun:
    spec: ProbeSpec
    rounds: tuple[Round, ...]
    backend: str                 # "scamper" | "replay:scamper"
    backend_version: str | None
    started_at: datetime
    finished_at: datetime
    interrupted: bool = False    # Ctrl-C stopped the run early
```

`RawRound` (backend output before parsing) lives in `probe/backends/base.py`; see
[architecture.md](../architecture.md#interfaces).

## Enrichment types (`models.py`, produced by `enrich/`)

```python
@dataclass(frozen=True, slots=True)
class SourceRef:
    name: str                    # "iptoasn" | "team-cymru-dns" | "peeringdb" | "dbip-lite-city" | "rdns"
    version: str | None          # dataset date / ETag / sync timestamp
    live: bool                   # True if fetched live during this run

@dataclass(frozen=True, slots=True)
class IxpInfo:
    ix_id: int; ix_name: str; city: str | None; country: str | None
    member_asn: int | None       # from exact netixlan IP match; None if prefix-only match
    member_name: str | None

@dataclass(frozen=True, slots=True)
class GeoInfo:
    basis: GeoBasis
    confidence: Confidence       # RDNS_HINT → MEDIUM; DB_CITY, DB_COUNTRY → LOW
    country: str | None          # ISO 3166-1 alpha-2
    city: str | None
    lat: float | None            # rounded to 1 decimal (≈11 km); never more precise
    lon: float | None
    source: SourceRef

@dataclass(frozen=True, slots=True)
class HopEnrichment:
    ip: IPAddress
    address_class: AddressClass
    origin_asns: tuple[int, ...]         # may be empty; >1 for MOAS
    as_name: str | None
    prefix: str | None                   # CIDR or "start-end" range as given by the source
    asn_source: SourceRef | None
    registry_country: str | None         # RIR country: NOT a location, never shown as one
    ixp: IxpInfo | None
    rdns: str | None
    geo: GeoInfo | None
    notes: tuple[str, ...] = ()          # e.g. "team-cymru timeout", "dataset older than 7 days"
```

## Analysis types (`analyze/`, all pure)

```python
@dataclass(frozen=True, slots=True)
class Interval:
    lo: float; hi: float

@dataclass(frozen=True, slots=True)
class HopStats:
    ttl: int
    kind: RowKind
    sent: int
    received: int
    loss: float                          # 1 - received/sent
    loss_ci: Interval                    # Wilson 95%
    loss_state: LossState
    rtt_min_ms: float | None
    rtt_median_ms: float | None
    rtt_median_ci_ms: Interval | None
    rtt_p90_ms: float | None
    jitter_ms: float | None
    reply_sequence: tuple[bool | None, ...]   # per round; None = no probe sent at this TTL
    runs_z: float | None
    responders: tuple[tuple[IPAddress, int], ...]   # sorted by count desc, then IP
    primary_responder: IPAddress | None
    mpls_labels_seen: bool
    flags: frozenset[HopFlag]

@dataclass(frozen=True, slots=True)
class HopAttribution:
    ttl: int
    effective_asn: int | None
    asn_basis: AsnBasis
    as_name: str | None
    ixp_crossing: Confidence | None      # None = no confirmed crossing at this hop
    geo_usable: bool                     # geo present, confidence ≥ MEDIUM, not GEO_IMPLAUSIBLE

@dataclass(frozen=True, slots=True)
class Location:
    kind: LocationKind
    from_hop: int | None                 # loss: last clean hop; latency: previous eligible hop p;
                                         # None for LOCAL / PATH_START
    to_hop: int                          # first degraded hop; MPLS_SEGMENT: segment's last hop
    from_asn: int | None
    to_asn: int | None
    ixp_name: str | None

@dataclass(frozen=True, slots=True)
class Finding:
    code: FindingCode
    hops: tuple[int, ...]
    message: str
    data: tuple[tuple[str, float | int | str], ...] = ()

@dataclass(frozen=True, slots=True)
class Caveat:
    code: CaveatCode
    message: str

@dataclass(frozen=True, slots=True)
class AnalysisResult:
    verdict: Verdict
    confidence: Confidence | None        # None for INSUFFICIENT_DATA, PATH_UNSTABLE, DESTINATION_UNREACHABLE
    summary: str                         # the one-sentence verdict
    location: Location | None            # set for LOSS_STARTS_AT / LATENCY_STARTS_AT
    secondary: Location | None           # latency onset when the main verdict is LOSS_STARTS_AT
    evidence: tuple[str, ...]
    hops: tuple[HopStats, ...]
    attributions: tuple[HopAttribution, ...]
    findings: tuple[Finding, ...]
    caveats: tuple[Caveat, ...]
    hints: tuple[str, ...]
    rounds_completed: int
    thresholds_overridden: tuple[tuple[str, float], ...]
```

## Recording format (`*.spa.json`)

This is written by `spa trace --record PATH` and read by `spa analyze PATH` and by `ReplayBackend`.
It stores the **raw** backend output of each round, so replays go through the same parser as live
runs ([ADR-0003](../decisions/0003-measurement-strategy.md)).

```json
{
  "format": "spa-recording",
  "format_version": 1,
  "tool_version": "0.1.0",
  "backend": "scamper",
  "backend_version": "<as printed by scamper, M0>",
  "spec": {"target": "example.net", "target_ip": "203.0.113.10", "protocol": "udp",
           "sport": 52731, "dport": 33435, "icmp_checksum": null,
           "first_ttl": 1, "max_ttl": 30, "rounds": 100, "interval_s": 1.0, "wait_s": 2,
           "gap_limit": 5, "round_timeout_s": 60.0, "quick": false},
  "started_at": "2026-10-13T09:00:00Z",
  "finished_at": "2026-10-13T09:01:43Z",
  "interrupted": false,
  "scrubbed": true,
  "rounds": [
    {"index": 0, "started_at": "2026-10-13T09:00:00Z", "duration_s": 0.84,
     "returncode": 0, "stdout": "<raw scamper JSON lines, verbatim>", "stderr": ""}
  ]
}
```

A file with an unknown `format_version` makes `spa analyze` exit with code 3.

## JSON output schema v1 (`spa trace --json`, `spa analyze --json`)

The schema is generated from the Pydantic models in `report/json_models.py` with
`model_json_schema(mode="serialization")` and committed as `schema/report-v1.schema.json`. A test
checks that the committed file matches the generated one.

Versioning: `schema_version` is `"MAJOR.MINOR"`. Adding an optional field bumps MINOR. Removing or
renaming a field, or changing its meaning or type, bumps MAJOR and needs a new ADR. Field names below
are the contract: do not rename them without asking.

Example (illustrative values; the addresses are RFC 5737 documentation ranges):

```json
{
  "schema_version": "1.0",
  "tool": {"name": "smart-path-analyzer", "version": "0.1.0"},
  "generated_at": "2026-10-13T09:01:44Z",
  "target": {"input": "example.net", "ip": "203.0.113.10",
             "flow": {"protocol": "udp", "sport": 52731, "dport": 33435, "icmp_checksum": null}},
  "measurement": {"backend": "scamper", "backend_version": "…", "rounds_requested": 100,
                  "rounds_completed": 100, "rounds_failed": 0, "interval_s": 1.0, "quick": false,
                  "interrupted": false, "started_at": "2026-10-13T09:00:00Z",
                  "finished_at": "2026-10-13T09:01:43Z"},
  "verdict": {
    "code": "LOSS_STARTS_AT",
    "confidence": "high",
    "exit_code": 1,
    "summary": "Loss starts inside AS3356 (Lumen), between hops 6 and 7.",
    "location": {"kind": "same_as", "from_hop": 6, "to_hop": 7,
                 "from_asn": 3356, "to_asn": 3356, "ixp_name": null},
    "secondary_location": null,
    "evidence": ["Destination loss 6.0% (6/100, 95% CI 2.8–12.5%); hop 6 0.0% (0/100); every hop from 7 to 10 shows comparable loss."],
    "caveats": [],
    "hints": ["Run with --rounds 300 to narrow the loss estimate."]
  },
  "hops": [
    {"ttl": 6, "kind": "router", "responder": "198.51.100.6", "other_responders": [],
     "rdns": "ae-1.r01.exmpl.example.net", "address_class": "public",
     "asn": 3356, "asn_basis": "origin", "origin_asns": [3356], "as_name": "Lumen",
     "ixp": null,
     "geo": {"basis": "rdns_hint", "confidence": "medium", "country": "US", "city": "Chicago",
             "lat": 41.9, "lon": -87.6, "source": "rdns"},
     "sent": 100, "received": 100, "loss": 0.0, "loss_ci": [0.0, 0.037], "loss_state": "clean",
     "rtt_ms": {"min": 11.0, "median": 12.0, "median_ci": [11.7, 12.4], "p90": 13.1, "jitter": 0.6},
     "runs_z": null, "flags": []}
  ],
  "findings": [
    {"code": "ICMP_DEPRIORITIZED", "hops": [4],
     "message": "Hop 4 loses 30.0% of its own replies but later hops do not; this is ICMP rate limiting, not forwarding loss.",
     "data": {"hop_loss": 0.3, "dest_loss": 0.06}}
  ],
  "data_sources": [
    {"name": "iptoasn", "version": "2026-10-13T08:00:00Z", "license": "PDDL-1.0", "attribution": null},
    {"name": "dbip-lite-city", "version": "2026-10", "license": "CC-BY-4.0",
     "attribution": "IP Geolocation by DB-IP (https://db-ip.com)"}
  ],
  "thresholds_overridden": {}
}
```

Rules: `loss` and `loss_ci` are fractions. `rtt_ms` fields are `null` when `received == 0`.
`verdict.location` is `null` unless the code is `LOSS_STARTS_AT` or `LATENCY_STARTS_AT`.
`verdict.confidence` is `null` for the three inconclusive codes. Snapshot tests exclude
`generated_at`, `measurement.started_at`, `measurement.finished_at`, `tool.version` and
`data_sources[*].version`.
