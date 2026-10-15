# Smart Path Analyzer (`spa`)

> **Status: design phase.** Nothing below is implemented yet. This README describes the planned
> v0.1.0. Progress is tracked in [docs/roadmap.md](docs/roadmap.md).

`spa` is a traceroute that tells you **where a network problem really is**, and when there isn't one.

```text
$ sudo -E spa trace example.net
…
Verdict: Loss starts inside AS3356 (Lumen), between hops 6 and 7.   [confidence: high]
```

```text
Verdict: No real problem; hop 4 just deprioritizes ICMP.   [confidence: high]
```

## Why it is different

mtr and traceroute show a loss column per hop. Most of the scary numbers there are not real.
Routers answer traceroute from a slow, rate-limited control plane, so a hop can "lose" 40% of its
own replies while forwarding every packet just fine. Seasoned operators read these outputs with one
rule: **loss or latency counts only if it continues to every later hop and the destination**
([APNIC](https://blog.apnic.net/2022/03/28/how-to-properly-interpret-a-traceroute-or-mtr/),
[Cloudflare](https://www.cloudflare.com/learning/network-layer/what-is-mtr/)). `spa` applies that rule for you:

- **Paris traceroute** keeps the flow identifiers fixed, so load balancing can't create fake paths
  ([Augustin et al.](https://arxiv.org/pdf/0904.2733)). Probing is done by CAIDA's
  [scamper](https://www.caida.org/catalog/software/scamper/).
- **Repeated rounds** (100 by default) with **confidence intervals**. `spa` says "insufficient data" instead
  of guessing from 10 probes, because 0/10 lost is still consistent with 28% loss.
- **Enrichment** of every hop with ASN, network name, IXP membership (PeeringDB), and approximate location
  (labeled with its source and confidence).
- **Honest verdicts**: one sentence, the evidence behind it, and caveats such as "the delay may be on the
  return path" or "the destination does not answer UDP probes".

It is not a monitoring system: there is one vantage point and one target per run, with no BGP
correlation and no web UI.

## Install (planned)

Requirements: Linux or macOS, Python ≥ 3.12, and scamper.

```bash
# scamper
sudo apt install scamper          # Debian/Ubuntu (package name to be confirmed; CAIDA also has a PPA)
brew install scamper              # macOS

# spa
uv tool install smart-path-analyzer     # or: pipx install smart-path-analyzer
spa data update                          # download offline ASN / IXP / geo data (no root needed)
```

## Privileges

Sending Paris probes needs raw sockets. scamper must start as root, or be setuid-root as its manual
recommends. It then drops privileges itself
([scamper(1)](https://www.caida.org/catalog/software/scamper/man/scamper.1.pdf)).

| Platform | How to run |
|---|---|
| Linux | `sudo -E spa trace <target>`, or make scamper setuid: `sudo chmod u+s "$(command -v scamper)"` |
| macOS | `sudo -E spa trace <target>` |
| Docker (Linux) | `docker run --rm --cap-add=NET_RAW --network host <image> spa trace <target>` (`--network host` avoids an extra NAT hop) |
| Windows | Not supported. WSL2 is untested (its NAT may add a hop and block replies) |

Whether `setcap cap_net_raw+ep` on scamper is enough is being checked ([roadmap M0](docs/roadmap.md#m0-spike-verify-scamper-12-days)).
`spa analyze` (replaying a recording) and `spa data …` never need root.

## Usage

```bash
sudo -E spa trace example.net                              # 100 rounds, about 2 minutes
sudo -E spa trace example.net --quick                      # 30 rounds: only large loss can be called
sudo -E spa trace example.net --protocol tcp --dport 443   # if the destination ignores UDP
sudo -E spa trace example.net --json > report.json         # machine-readable, schema v1
sudo -E spa trace example.net --record run.spa.json        # save the raw run
spa analyze run.spa.json                                   # re-analyze later, no root
spa data status                                            # dataset versions and licenses
```

Sample output (illustrative; real output will replace it in M6):

```text
spa → example.net (203.0.113.10)  udp-paris sport 52731 dport 33435  100 rounds

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

### Verdicts and exit codes

| Verdict | Meaning | Exit |
|---|---|---|
| `NO_PROBLEM` | Destination loss and latency are fine | 0 |
| `ICMP_DEPRIORITIZATION_ONLY` | Some hops look bad, but only in their own replies | 0 |
| `LOSS_STARTS_AT` | Loss begins at a link and persists to the destination | 1 |
| `LATENCY_STARTS_AT` | A latency increase begins at a link and persists | 1 |
| `INSUFFICIENT_DATA` | Too few probes to decide; run longer | 2 |
| `DESTINATION_UNREACHABLE` | The destination never answered this probe type | 2 |
| `PATH_UNSTABLE` | The route changed during the run | 2 |
| usage error / probing or permission error | | 3 / 4 |

## Data sources and attribution

- **IP to ASN:** [iptoasn.com](https://iptoasn.com/), public domain (ODC PDDL v1.0). Live fallback:
  [Team Cymru IP to ASN](https://www.team-cymru.com/ip-asn-mapping) (DNS).
- **IXPs:** [PeeringDB](https://www.peeringdb.com/). Data is synced at most once a day within the
  [API limits](https://docs.peeringdb.com/howto/work_within_peeringdbs_query_limits/).
  An optional `SPA_PEERINGDB_API_KEY` raises the limit.
- **Geolocation:** IP Geolocation by [DB-IP](https://db-ip.com), licensed under
  [CC BY 4.0](https://db-ip.com/db/lite.php). Router locations from free databases are often wrong, so
  `spa` shows them as approximate (`~City`) and checks them against RTT physics.
- **Probing:** [scamper](https://www.caida.org/catalog/software/scamper/) by CAIDA (GPLv2). It is run as a
  separate program and is not bundled or linked.

## Development

See [AGENTS.md](AGENTS.md) for commands and rules, and [docs/](docs/) for the architecture, specs, ADRs
and roadmap.

## License

MIT
