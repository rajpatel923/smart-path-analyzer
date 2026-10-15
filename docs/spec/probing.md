# Spec: probing (scamper integration)

Rationale: [ADR-0002](../decisions/0002-probing-backend.md) (why scamper as a subprocess) and
[ADR-0003](../decisions/0003-measurement-strategy.md) (why repeated rounds). Types:
[data-model.md](data-model.md). Interfaces: [architecture.md](../architecture.md#interfaces).

> **READ THIS FIRST.** The research did not run scamper. The **only** scamper facts confirmed from
> the man page are the `trace` options `-P` (method: UDP, ICMP, UDP-Paris, ICMP-Paris, TCP,
> TCP-ACK; UDP-Paris is the default), `-q` (attempts, default 2), `-w` (wait timeout, default 5 s)
> and `-W` (wait between probes, in tens of ms), plus the daemon options `-U`/`-P`/`-D`
> ([scamper(1)](https://www.caida.org/catalog/software/scamper/man/scamper.1.pdf)).
> Everything else on this page is **UNVERIFIED — confirm in M0**: the top-level flags that select a
> command, target and JSON output, the flags for source port, destination port, first TTL, max TTL
> and gap limit, and every JSON field name. Do not write the parser until the
> [M0 checklist](#m0-spike-checklist) is complete and real JSON is committed under
> `tests/fixtures/scamper/`. Never fill a gap by guessing. Write `TODO(M0)` and stop.

## Invocation model

- **One scamper process per round.** Each round is one complete Paris trace of the target. It is
  started with `asyncio.create_subprocess_exec(*argv, stdout=PIPE, stderr=PIPE)`: an argv list,
  never a shell.
  <!-- Why: per-round processes keep failures isolated and make every round's raw stdout a self-contained fixture. A long-lived daemon (-U socket) is a possible optimization only if M0 shows spawn overhead matters. -->
- Output is read with `await asyncio.wait_for(proc.communicate(), timeout=round_timeout_s)`. On
  timeout: `proc.kill()`, `await proc.wait()`, record the round as `RoundStatus.TIMEOUT`.
  `communicate()` avoids pipe-buffer deadlocks. `wait_for` cancellation does not kill the child,
  so the kill must be explicit ([Python asyncio subprocess docs](https://docs.python.org/3/library/asyncio-subprocess.html)).
- The target is resolved once, before round 0, to a single IPv4 address. That address goes to
  scamper, so DNS round-robin cannot change the destination between rounds.
- The argv is built in exactly one function, `probe/backends/scamper.py::build_argv(spec) -> list[str]`,
  and unit-tested against a snapshot. **Candidate** shape (UNVERIFIED, M0 replaces this):

  ```text
  scamper -O json -i <target_ip> -c "trace -P udp-paris -q 1 -w <wait_s> -s <sport> -d <dport> -f <first_ttl> -m <max_ttl> -g <gap_limit>"
  ```

  `-P udp-paris`, `-q 1` and `-w` are man-page confirmed. `-O json`, `-i`, `-c`, `-s`, `-d`, `-f`, `-m`
  and `-g` are guesses that M0 must confirm or replace. Note that `-P` means "method" inside a
  `trace` command but "control port" at the top level.
- `-q 1`: one attempt per TTL per round. Per-hop loss comes from many rounds, not from attempts
  within one round ([ADR-0003](../decisions/0003-measurement-strategy.md)).

## Flow fixing

| Protocol | Fixed per run | Probe ID carried in (done by scamper) |
|---|---|---|
| `udp` (default) | sport (random in 49152–65535, chosen once per run), dport (default 33435) | UDP checksum, adjusted via payload |
| `icmp` | ICMP checksum | ICMP sequence, with identifier offset to keep the checksum constant |
| `tcp` | sport (random, chosen once), dport (default 443) | TCP sequence number |

Probe-ID placement is from [Augustin et al.](https://arxiv.org/pdf/0904.2733). Load balancers hash
the five-tuple, TOS, and the ICMP code and checksum. A fixed tuple therefore keeps every probe of every round on
one ECMP branch. The project never builds packets itself.

UNVERIFIED (M0): the scamper flag that pins the ICMP checksum (the Python API has `icmp_sum`); the
default UDP-Paris dport; whether the same `sport` given to separate scamper processes is honored.

**Flow check (parser):** every round's reported sport, dport and ICMP checksum (JSON field names
UNVERIFIED) must equal `spec.flow`. A mismatch raises `FlowMismatchError`, which fails the run with
exit 4. Analysing such a run would silently mix ECMP paths.

## Rounds and timing

| Setting | Default | Notes |
|---|---|---|
| `rounds` | 100 | `--quick` sets 30 and adds caveat `QUICK_MODE` |
| `interval_s` | 1.0 | Minimum start-to-start spacing. Allowed 0.5–10.0; below 1.0 prints a rate-limit warning |
| `wait_s` | 2 | Per-probe reply timeout (scamper `-w`). Allowed 1–5 |
| `first_ttl` / `max_ttl` | 1 / 30 | |
| `gap_limit` | 5 | Consecutive silent TTLs before scamper stops the round. Flag UNVERIFIED |
| `round_timeout_s` | 60.0 | Hard kill per round |
| `MAX_CONSECUTIVE_FAILED_ROUNDS` | 3 | Then abort with exit 4 |

Rounds run **sequentially**. Round i+1 starts at `max(start_i + interval_s, end_i)`.
<!-- Why: overlapping rounds would run two processes on the same flow tuple; reply matching across processes is unverified. -->
Expected duration is about rounds × max(interval, round duration). A silent stretch near the end of the path
can push a round to around (gap_limit × wait_s) seconds. M0 measures this on a real path.

Rate-limit hygiene: one probe per hop per round, about 1 per second, matches mtr, trippy and scamper
ping defaults. Linux hosts space rate-limited ICMP errors (Time Exceeded, Dest Unreachable) by
`icmp_ratelimit` = 1000 ms per target by default
([kernel ip-sysctl](https://docs.kernel.org/networking/ip-sysctl.html)). Probing faster makes the
tool create the very artifacts it is meant to dismiss
([Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf)).
Only one `spa trace` should run per host at a time. Concurrent multi-target runs are out of scope.

Ctrl-C: stop after the current round. If at least `MIN_ROUNDS_ABSOLUTE` rounds completed, analyze
them with `interrupted=true`. Otherwise exit 2 (`INSUFFICIENT_DATA`). A second Ctrl-C kills the child
and exits 130.

## Parsing (`probe/parse/scamper_json.py`)

`parse_round(raw: RawRound, spec: ProbeSpec) -> Round` is a pure function (string in, dataclass out).
It must:

1. Split stdout into JSON lines and keep only the trace record. M-Lab saw `cycle-start`, `tracelb`
   and `cycle-stop` lines for MDA output
   ([M-Lab](https://www.measurementlab.net/blog/2022-extracting-single-traceroute/)). The equivalent
   for `trace` is UNVERIFIED.
2. Map each reply to `ProbeReply`. Candidate fields, taken from the **Python module's**
   `ScamperTraceHop` attributes (the JSON names may differ, UNVERIFIED): `probe_ttl`, `src`/address,
   `rtt`, `reply_ttl`, `attempt`, `icmp_type`, `icmp_code`, `icmp_q_ttl`, `icmp_exts`
   ([scamper Python docs](https://www.caida.org/catalog/software/scamper/python/_sources/index.rst.txt)).
   Confirm the RTT unit and encoding in M0. Convert to float ms.
3. Set `ReplyKind`: TIME_EXCEEDED for ICMP type 11; DESTINATION when the responder is the target and
   the reply ends the trace (UDP: port unreachable; ICMP: echo reply; TCP: SYN-ACK or RST);
   UNREACHABLE for other ICMP type 3 from a non-target address; OTHER otherwise.
4. Derive `probed_ttls` (needed for per-hop `sent`). Preferred: the JSON lists unanswered probes.
   Fallback: `first_ttl..last TTL probed`, inferred from the stop reason and hop count. **M0 decides
   which. The classifier's loss math depends on it.**
5. Keep at most one reply per TTL. Drop duplicates but count them in a debug log.
6. Parse RFC 4950 MPLS label stacks into `MplsLabel` if present (representation UNVERIFIED).
7. Raise `ParseError` with the round index and a 200-character excerpt on anything unexpected.
   Never guess.

Parser tests run on committed real outputs only ([testing.md](../testing.md#fixtures)).

## Privileges and preflight

scamper "must be started as root" for most modules, and the man page recommends setting the setuid
bit. It then drops privileges to `nobody` in a chroot
([scamper(1)](https://www.caida.org/catalog/software/scamper/man/scamper.1.pdf)).

`ScamperBackend.preflight()` runs before round 0 and maps problems to exit 4 with these messages:

| Check | Failure message (stderr) |
|---|---|
| `shutil.which("scamper")` | `scamper not found on PATH. Install it: Debian/Ubuntu: apt install scamper (or the CAIDA PPA); macOS: brew install scamper. See README → Install.` |
| Effective uid 0, **or** scamper binary is setuid-root | `scamper needs root. Use one of: sudo -E spa trace …; sudo chmod u+s "$(command -v scamper)" (setuid, as scamper's man page recommends); or Docker (see README).` |
| One trial trace to `first_ttl` succeeds (live permission check) | `scamper could not open raw sockets: <first stderr line>.` |

UNVERIFIED (M0): whether `setcap cap_net_raw+ep` on scamper is enough without root (its privsep and
chroot may need more); the exact stderr and exit status when run unprivileged; the apt package name.
Until M0 confirms setcap, the docs recommend only sudo, setuid or Docker.

Platforms:
- **Linux**: supported.
- **macOS**: supported (`brew install scamper`, run with `sudo`).
- **Docker**: `docker run --rm --cap-add=NET_RAW --network host <image> spa trace <target>`.
  `--network host` avoids an extra NAT hop at TTL 1. Docker's defaults include NET_RAW, but
  Podman/RHEL builds removed it, hence the explicit flag
  ([CIS Docker 5.3](https://docs.datadoghq.com/security/default_rules/cis-docker-1.2.0-5.3)). Do not
  add `--cap-drop=ALL` until M0 shows scamper's privsep works without SETUID, SETGID and SYS_CHROOT.
  Docker Desktop on macOS and Windows adds a VM NAT hop, which is a known artifact.
- **Windows**: unsupported. **WSL2**: untested; NAT mode adds a hop and Time Exceeded may not come
  back (research gap).

## Path-change detection

The probe layer records every responder per TTL per round and never merges or drops them. Detection
and the `PATH_UNSTABLE` verdict are classifier logic ([classifier.md §5.2](classifier.md#52-path-change-analyzepathpy)).
The Rich report shows the alternate responders for unstable TTLs.

## Recording fixtures

```bash
sudo -E uv run spa trace example.net --rounds 30 --record tests/fixtures/recordings/<scenario>.spa.json
uv run spa analyze tests/fixtures/recordings/<scenario>.spa.json      # must reproduce the verdict
```

- Recordings are **scrubbed by default**: every occurrence of the probing host's source address in
  the raw stdout is replaced with `192.0.2.1` (RFC 5737), and `"scrubbed": true` is set. `--no-scrub`
  keeps raw data and must never be used for committed fixtures. Before committing, also check that hop 1–2 addresses and rDNS do not
  identify a home network. Replace them by hand with 192.0.2.x if needed.
- Single-round raw outputs for parser tests go in `tests/fixtures/scamper/<case>.jsonl`. Each one comes
  with a `<case>.md` noting the scamper version, the exact argv, the OS and what the case shows.
- Name recordings after what they show (`clean_cloudflare_udp.spa.json`, `dst_silent_udp.spa.json`),
  not after when they were made.

## M0 spike checklist

Do these on Linux (and macOS if available) before any parser code. Record results in
`docs/spec/probing.md` (rewrite every unverified note as fact) and commit the raw outputs.

- [ ] scamper version string and how to print it. Record install method and package name.
- [ ] The exact argv that runs one `trace` to an IP and prints JSON to stdout, with UDP-Paris, one
      attempt, fixed sport and dport, max TTL, first TTL and gap limit. Confirm each flag in `man scamper`.
- [ ] stdout line structure (record types per line) and scamper's exit status on success.
- [ ] JSON field names and types: responder address, probe TTL, RTT (unit and encoding), reply TTL,
      ICMP type and code, quoted TTL, attempt, sport, dport, ICMP checksum, stop reason, hop count, start time.
- [ ] Are unanswered probes present in the JSON? If not, what tells us the last TTL probed?
- [ ] Behavior when the destination probe is lost: does the trace continue to TTL+1?
- [ ] Gap limit default, its flag, and the stop reason when it triggers.
- [ ] The same `sport` passed to two consecutive scamper processes shows up in both outputs.
- [ ] ICMP-Paris: how the checksum is chosen or pinned and how it is reported. TCP: dport flag and
      what a SYN-ACK or RST from the target looks like.
- [ ] RFC 4950 MPLS extension representation (find a path with labels if possible).
- [ ] Round duration on a clean path and on a path with a silent destination (wait 2, gap 5).
      Process spawn overhead.
- [ ] Unprivileged run: exit status and stderr. Setuid run works. `setcap cap_net_raw+ep` alone works or not.
      Docker with `--cap-add=NET_RAW --network host` works.
- [ ] Commit at least: clean path, silent destination, a path with `*` hops, ICMP-Paris, and TCP to 443,
      in `tests/fixtures/scamper/`. (Full `.spa.json` recordings need `spa trace --record`, so they come in M1.)
