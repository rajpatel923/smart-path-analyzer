---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0002: scamper as a subprocess with JSON output is the probing backend

## Context and Problem Statement

Per-hop loss and latency are only meaningful when every probe follows the same path. Per-flow
load balancers hash the five-tuple, TOS, and the ICMP code and checksum. Classic traceroute changes
those fields on every probe, which creates fake loops and diamonds. Paris traceroute removed 89.8% of
loop signatures and 52.6% of diamonds in its authors' dataset
([Augustin et al.](https://arxiv.org/pdf/0904.2733)). We need a Paris-correct engine that is
maintained, testable without root, and licensable alongside MIT code.

## Decision Drivers

- Paris correctness for UDP, ICMP and TCP without writing packet code ourselves.
- License separation: the project is MIT.
- Easy installs: no C-extension builds.
- Raw output that can be recorded once and replayed in CI.
- Maintained in 2026.

## Considered Options

1. **scamper CLI as a subprocess, JSON output (chosen)**
2. scamper through its official Python module (`ScamperCtrl`)
3. Scapy, implementing Paris ourselves
4. trippy (Rust) as a subprocess
5. mtr, icmplib, Dublin Traceroute, libparistraceroute

## Pros and Cons of the Options

### 1. scamper CLI subprocess
- Good: the `trace` command defaults to UDP-Paris and also offers ICMP-Paris and TCP
  ([scamper(1)](https://www.caida.org/catalog/software/scamper/man/scamper.1.pdf)). We get correct
  Paris behavior for free.
- Good: actively maintained (release cvs-20260902) and packaged for Debian/Ubuntu (PPA), Homebrew and
  MacPorts ([CAIDA](https://www.caida.org/catalog/software/scamper/)). M-Lab and CAIDA Ark run it at scale.
- Good: under the GPL FAQ, fork/exec plus parsing output is communication between separate programs.
  scamper's GPLv2 does not reach our MIT code
  ([GPLv2 FAQ mirror](https://www.southampton.ac.uk/~qs2g22/mirror/www.gnu.org/licenses/old-licenses/gpl-2.0-faq.html)).
- Good: stdout JSON is trivially saved as a fixture and replayed through the same parser.
- Bad: needs root or setuid. Process spawn per round. Tied to the JSON format, which scamper itself
  says is "better to use warts" for, and which we have **not** seen yet. The flags and field names are
  unverified. This is why M0 exists.

### 2. scamper Python module
- Good: typed result objects (`ScamperTrace`, `ScamperTraceHop`), `do_trace(..., attempts, all_attempts, sport, dport)`
  ([Python docs](https://www.caida.org/catalog/software/scamper/python/)).
- Bad: the module is GPL-2.0-only and imported in-process, so the combined-work concern applies.
- Bad: it needs the C libraries libscamperctrl and libscamperfile plus headers to build
  ([PyPI scamper](https://pypi.org/project/scamper/)), and a separately run root daemon.

### 3. Scapy
- Good: pure Python packet crafting; good for learning. Version 2.8.0 is maintained.
- Bad: we would own Paris checksum tricks, reply matching via quoted headers, timing and rate control.
  Kernel RSTs break TCP probes. pcap round-trip jitter affects RTTs
  ([Scapy troubleshooting](https://scapy.readthedocs.io/en/latest/troubleshooting.html)).
- Bad: GPL-2.0-only and imported in-process ([PyPI scapy](https://pypi.org/pypi/scapy/json)), which pushes the project toward GPL.

### 4. trippy subprocess
- Good: polished; JSON report over N cycles; Paris and Dublin strategies; Windows support; Apache-2.0
  ([Trippy CLI](https://trippy.rs/reference/cli/)).
- Bad: Paris/Dublin are UDP only. The JSON gives per-hop summaries rather than per-probe data, and the
  schema is unverified. No release since 2025-05-05 ([crates.io](https://crates.io/crates/trippy)).
  Paris needs raw sockets anyway ([privileges](https://trippy.rs/guides/privileges/)).
- Kept as a possible second backend (stretch goal).

### 5. Others
- mtr: no Paris mode. UDP mode puts the sequence number in the port, which changes the flow on every
  probe ([mtr man](https://github.com/traviscross/mtr/blob/master/man/mtr.8.in)). Useful only for sanity comparison.
- icmplib: ICMP only, no Paris, traceroute needs root, last release 2023-10-10 ([GitHub](https://github.com/ValentinBELYN/icmplib)).
- Dublin Traceroute: Python bindings last released 2017-10-15 ([GitHub](https://github.com/insomniacslk/dublin-traceroute)).
- libparistraceroute / paris-traceroute: no sign of recent maintenance ([GitHub](https://github.com/libparistraceroute/libparistraceroute)).

## Decision Outcome

Chosen: **option 1**. `ScamperBackend` runs one `scamper` process per round with
`asyncio.create_subprocess_exec` (an argv list, no shell), reads JSON from stdout with `communicate()`,
and explicitly kills the child on timeout. Backends sit behind the `ProbeBackend` Protocol with
capability flags ([architecture.md](../architecture.md#interfaces)). `ReplayBackend` feeds recorded raw
output through the same parser. We never import scamper's Python module or Scapy.

**UNVERIFIED — confirm in M0:** the exact top-level flags for JSON output and a single target, and the
`trace` flags for sport, dport, first and max TTL and gap limit, plus every JSON field name. Only
`-P`, `-q`, `-w` and `-W` are confirmed from the man page. Agents must not invent flags or fields
([probing.md](../spec/probing.md)).

### Consequences

- Good: no packet code, no C builds, clean license boundary, deterministic replay tests.
- Bad: users must install scamper and run as root or setuid. macOS works via Homebrew. Windows is unsupported.
- Bad: if M0 shows scamper's JSON lacks something the classifier needs (for example unanswered probe
  TTLs), we must derive it or supersede this ADR (fallback: warts plus `sc_warts2json`, or trippy).

### Confirmation

- No `import scamper` or `import scapy` anywhere: a test greps `src/` for both.
- `build_argv` is snapshot-tested against the flags verified in M0.
- Parser tests run only on committed real scamper output.

## More Information

[probing_engine research](../research.md#probing-and-paris-traceroute). Related: [ADR-0003](0003-measurement-strategy.md).
