# Spec: CLI

Framework: Typer (`~=0.27`) + Rich, in `src/smart_path_analyzer/cli.py`, entry point
`spa = "smart_path_analyzer.cli:main"`. Rationale: [ADR-0001](../decisions/0001-language-and-tooling.md),
[ADR-0008](../decisions/0008-output-and-exit-codes.md). Do not change flag names, defaults or exit
codes without asking.

## Commands

```text
spa trace <target>              Probe, enrich, classify, report
spa analyze <recording.spa.json> Re-enrich and re-classify a recorded run (no root needed)
spa data update                 Download / refresh offline datasets
spa data status                 Show dataset versions, ages, licenses, cache location
spa --version
```

### `spa trace <target>`

`<target>` is a hostname or IPv4 address. A hostname is resolved once, and the first A record is
used and printed. IPv6 targets exit 3 with "IPv6 is not supported yet" (stretch goal).

| Flag | Default | Meaning |
|---|---|---|
| `--rounds N` | 100 | Number of Paris trace rounds. Allowed 1–3600; values below 20 always give `INSUFFICIENT_DATA` |
| `--quick` | off | Same as `--rounds 30`, plus caveat `QUICK_MODE`. Mutually exclusive with `--rounds` |
| `--interval S` | 1.0 | Minimum seconds between round starts. Allowed 0.5–10. Below 1.0 prints a warning |
| `--protocol {udp,icmp,tcp}` | udp | Paris variant (`udp-paris`, `icmp-paris`, `tcp` in scamper) |
| `--dport P` | 33435 (udp), 443 (tcp) | Fixed destination port. Error (3) with `icmp` |
| `--sport P` | random 49152–65535 | Fixed source port for the whole run |
| `--max-ttl N` | 30 | 1–64 |
| `--first-ttl N` | 1 | 1–max-ttl |
| `--wait S` | 2 | Per-probe reply timeout, integer seconds, 1–5 |
| `--json` | off | Print only the JSON report (schema v1) on stdout |
| `--record PATH` | none | Also write a `.spa.json` recording (scrubbed) |
| `--no-scrub` | off | Keep the source address in the recording. Never use for committed fixtures |
| `--offline` | off | No live DNS lookups (Team Cymru, rDNS). Local datasets only |
| `--cache-dir PATH` | platformdirs | Also `SPA_CACHE_DIR` |
| `--config PATH` | platformdirs config | TOML config file ([architecture.md](../architecture.md#configuration)) |
| `--no-color` | auto | Also honors `NO_COLOR` |
| `-v / --verbose` | off | Show the per-hop findings table and debug info on stderr |

The default dport 33435 is our choice: a fixed, conventional traceroute-range port. Whether
scamper's UDP-Paris default matches is UNVERIFIED (M0).

### `spa analyze <recording>`

Reads a recording and replays it through `ReplayBackend` and the same parser. Then it enriches with
the **current** local data and classifies. It takes `--json`, `--offline`, `--cache-dir`,
`--config` and `--no-color`. The verdict code for a recording only depends on recording + thresholds,
except for the `DETOUR` case ([enrichment.md](enrichment.md#failure-modes-degrade-never-crash)). A
missing file, invalid JSON or an unknown `format_version` exits 3.

### `spa data update`

| Flag | Default | Meaning |
|---|---|---|
| `--only {iptoasn,peeringdb,dbip}` | all | Repeatable |
| `--force` | off | Ignore minimum refresh intervals |
| `--cache-dir PATH` | platformdirs | |

It prints one line per dataset (`updated`, `not modified`, `skipped: fresh`, `failed: <reason>`).
Exit 0 if all requested datasets are usable afterwards, 4 if any failed and has no previous copy.
This command does not need root.

### `spa data status`

A table of name, version, fetched-at, age, license, attribution and path, plus the cache dir.
Always exits 0. `--json` gives the same data as JSON (not part of schema v1).

## Exit codes

| Code | Meaning | When |
|---|---|---|
| 0 | No problem found | `NO_PROBLEM`, `ICMP_DEPRIORITIZATION_ONLY`. `spa data …` success |
| 1 | Problem found | `LOSS_STARTS_AT`, `LATENCY_STARTS_AT` |
| 2 | Inconclusive | `INSUFFICIENT_DATA`, `DESTINATION_UNREACHABLE`, `PATH_UNSTABLE`. Ctrl-C before 20 rounds |
| 3 | Usage or input error | bad flags or values, unknown target, IPv6 target, bad recording file, bad config |
| 4 | Probing, permission or environment error | scamper missing, not root, raw socket failure, flow mismatch, 3 consecutive failed rounds, dataset download failure with no fallback |
| 130 | Interrupted | second Ctrl-C |

<!-- Why: a script can tell "the network is fine" (0) from "the network is broken" (1) from "we can't tell" (2) from "you/your machine are the problem" (3, 4). Inconclusive verdicts are not failures of the tool. -->

**Implementation note.** Click/Typer exits with status **2** on usage errors by default. That would
collide with "inconclusive". `main()` must run the Typer app with `standalone_mode=False` and
map exceptions itself. Usage errors become 3, `SpaError` subclasses become their `.exit_code`, and
`KeyboardInterrupt` becomes 130. Only `cli.py` calls `sys.exit`. With Typer vendoring Click since
0.26, which exception classes to catch is UNVERIFIED. Check against the installed Typer, and pin the
behavior with a test (`spa trace --rounds abc` must exit 3).

## Output

**Human (default).** Rich, written to stdout. Progress (rounds completed, a live loss column) goes to stderr
and only when stderr is a TTY. Layout:

```text
spa → example.net (203.0.113.10)  udp-paris sport 52731 dport 33435  100 rounds  scamper <version>

 Hop  Address          rDNS / Network                       Loss   (95% CI)     Min   Med   P90  Note
   1  192.168.1.1      (private)                            0.0%  (0.0–3.7)     0.5   0.7   1.1
   2  100.64.0.1       (CGNAT, likely your ISP)             0.0%  (0.0–3.7)     8.1   8.9  10.2
   3  198.51.100.3     AS7922 Comcast                       0.0%  (0.0–3.7)     9.0   9.8  11.0
   4  198.51.100.4     AS3356 Lumen  ~Chicago, US          30.0%  (21.9–39.6)  10.1  10.9  12.5  ICMP deprioritized
   5  198.51.100.5     AS3356 Lumen                         0.0%  (0.0–3.7)    10.6  11.4  12.9
   6  198.51.100.6     AS3356 Lumen  ae-1.chi (Chicago)     0.0%  (0.0–3.7)    11.0  12.0  13.1
   7  198.51.100.7     AS3356 Lumen                         7.0%  (3.4–13.8)   11.9  12.8  14.0  ← loss starts
   …
  10  203.0.113.10     AS64500 Example Net                  6.0%  (2.8–12.5)   14.0  15.1  17.3  destination

Verdict: Loss starts inside AS3356 (Lumen), between hops 6 and 7.   [confidence: high]
Evidence: Destination loss 6.0% (6/100, 95% CI 2.8–12.5%); hop 6 0.0% (0/100); every hop from 7 to 10 shows comparable loss.
Also: Hop 4 loses 30.0% of its own replies but later hops do not; this is ICMP rate limiting, not forwarding loss.
To raise confidence: run with --rounds 300; trace from the far end.
Data: iptoasn 2026-10-13 · PeeringDB 2026-10-13 · IP Geolocation by DB-IP
```

- Silent rows show `*`. Rows with alternate responders get a second, dim line per responder.
- Colors: verdict line green (exit 0), red (1), yellow (2). Meaning is never carried by color alone.
- The `Data:` line always includes the DB-IP attribution when DB-IP data was used (CC BY 4.0).
- Rendered output is snapshot-tested at a fixed width of 120 with color off.

**JSON (`--json`).** Exactly one JSON document on stdout (schema v1, [data-model.md](data-model.md#json-output-schema-v1-spa-trace---json-spa-analyze---json)),
with a trailing newline and nothing else. Warnings and progress go to stderr. `verdict.exit_code` equals the process exit code.
For exits 3 and 4 there is no report. stdout is empty, and stderr has a one-line message plus a hint.

## Examples

```bash
sudo -E spa trace example.net                              # 100 rounds, ~2 minutes
sudo -E spa trace 203.0.113.10 --quick                     # 30 rounds, large loss only
sudo -E spa trace example.net --protocol tcp --dport 443   # when the destination ignores UDP
sudo -E spa trace example.net --json > report.json; echo $?
sudo -E spa trace example.net --record run.spa.json
spa analyze run.spa.json                                   # no root
spa data update && spa data status
```
