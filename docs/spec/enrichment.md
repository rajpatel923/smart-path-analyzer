# Spec: enrichment

Rationale: [ADR-0004](../decisions/0004-ip-to-asn-source.md) (ASN),
[ADR-0005](../decisions/0005-ixp-detection-peeringdb.md) (IXP),
[ADR-0006](../decisions/0006-geolocation.md) (geo). Output type: `HopEnrichment` in
[data-model.md](data-model.md#enrichment-types-modelspy-produced-by-enrich).

Principle: **offline-first**. `spa data update` downloads the datasets. `spa trace` and `spa analyze`
look up locally. The only live lookups are Team Cymru DNS for ASN misses and rDNS, and both are
cached. `--offline` disables both. Every field records its source.
<!-- Why: every live source is throttled (PeeringDB 20/min, Team Cymru null-routes heavy whois) or has shut down before (iptoasn API retired 2020). -->

## Lookup order per hop

```
1. Classify address (special-range table)       → non-PUBLIC: stop; no external lookups
2. PeeringDB netixlan exact-IP match             → ixp + member_asn
3. PeeringDB ixpfx longest-prefix match          → ixp (member_asn None)
4. iptoasn range lookup (bisect)                 → origin_asns, as_name, registry_country, prefix
5. Team Cymru DNS (only if 4 missed; not with --offline)
6. rDNS PTR (not with --offline)                 → rdns, then rDNS geo hint
7. DB-IP Lite City MMDB                          → geo (if no rDNS hint)
```

Enrichment runs **after** probing finishes, once per unique responder IP, with at most 8
concurrent DNS queries (`asyncio.Semaphore(8)`). It never slows down probe rounds.

## Special-range table (`enrich/special.py`)

Explicit table, checked in order, first match wins. Do **not** use `ipaddress.is_private` or
`is_global`: they changed in Python 3.13, and `100.64.0.0/10` is neither private nor global
([Python ipaddress docs](https://docs.python.org/3/library/ipaddress.html)). Ranges come from the
[IANA IPv4](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml)
and [IPv6](https://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xhtml)
special-purpose registries (updated 2025-10-09).

| Prefix | AddressClass |
|---|---|
| 0.0.0.0/8, ::/128 | SPECIAL_OTHER |
| 127.0.0.0/8, ::1/128 | LOOPBACK |
| 100.64.0.0/10 | CGNAT (checked before RFC 1918) |
| 169.254.0.0/16, fe80::/10 | LINK_LOCAL |
| 192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24, 2001:db8::/32, 3fff::/20 | DOCUMENTATION |
| 198.18.0.0/15, 2001:2::/48 | BENCHMARK |
| 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, fc00::/7 | PRIVATE |
| 240.0.0.0/4, 255.255.255.255/32 | RESERVED |
| 224.0.0.0/4, ff00::/8 | MULTICAST |
| 192.0.0.9/32, 192.0.0.10/32 | PUBLIC (globally reachable exceptions; listed before the /24) |
| 192.0.0.0/24, 64:ff9b:1::/48, 100::/64, 2001::/23, 5f00::/16 | SPECIAL_OTHER |
| everything else | PUBLIC |

IPv4-mapped IPv6 (`::ffff:0:0/96`) is normalized with `.ipv4_mapped` before matching. A unit test
pins one address per row. Treatment of 2002::/16 (6to4) and 64:ff9b::/96 (NAT64, globally reachable)
is a stretch-goal IPv6 decision: they are PUBLIC for now.

Display: non-PUBLIC hops show `(private)`, `(CGNAT, likely your ISP)` and so on, never "unknown ASN".
Neighbor-based AS inference is classifier logic ([classifier.md §6](classifier.md#6-attribution-analyzeattributionpy)).

## Sources

### iptoasn.com: primary IP→ASN and AS name
- **What:** `ip2asn-combined.tsv.gz` (v4 and v6). Columns: `range_start, range_end, AS_number,
  country_code, AS_description`. **License:** public domain (ODC PDDL v1.0). Updated hourly
  ([iptoasn.com](https://iptoasn.com/)). Its public API was retired on 2020-12-31, so use the file only.
- **Download URL:** UNVERIFIED. Copy it from the iptoasn.com page in M3 and store it as a constant
  with a comment giving the date checked.
- **Update:** `spa data update` with `If-None-Match`/`If-Modified-Since`; at most once per 24 h unless
  `--force`. <!-- Why: hourly upstream, but route changes that matter for a diagnosis are rarer; be polite. -->
- **Load:** at update time, convert to `datasets/iptoasn.sqlite` (table
  `ranges(family INT, start BLOB, end BLOB, asn INT, cc TEXT, descr TEXT)`, 16-byte big-endian ints
  so v4 and v6 sort correctly). At lookup time, load the rows for the needed family into two sorted
  arrays and use `bisect_right`. Rows with `AS_number == 0` mean "not routed" and count as a miss.
  The ranges are non-overlapping, which makes bisect exact.
  <!-- Why: pure Python, no compiled deps (pyasn is stale and sdist-only; pytricia sdist-only). Parse cost is paid once per update, not per run. -->
- **MOAS:** one range has one ASN. Extra origins come only from Team Cymru. Store a tuple anyway.

### Team Cymru DNS: live fallback for iptoasn misses
- `dnspython` TXT queries: IPv4 reversed octets under `origin.asn.cymru.com`, IPv6 reversed nibbles
  under `origin6.asn.cymru.com`. AS name from `AS<n>.asn.cymru.com` TXT. NXDOMAIN means no data
  ([Team Cymru](https://www.team-cymru.com/ip-asn-mapping)).
- The TXT answer is pipe-delimited. The exact field order for each zone is UNVERIFIED for DNS (the
  docs list whois verbose order). Confirm in M3 with `dig` and a recorded fixture.
- No port-43 whois per IP (they null-route heavy whois users). Timeout 2 s, no retries. Never more
  than 64 queries per run.
- Multiple origin ASNs are split on spaces into `origin_asns`. The returned country is registry data,
  never a location.

### PeeringDB: IXP detection
- Bulk-fetch `ix`, `ixlan`, `ixpfx` and `netixlan` with `depth=0` from
  `https://www.peeringdb.com/api/<object>`. Optionally `net` for names (≤ 5 requests per sync).
- **Rate limits:** 20 queries/min anonymous, 40/min with an API key; identical anonymous repeats
  2/min (1/hour if > 100 KB). Leave ≥ 2 s between requests and send an identifying User-Agent
  ([PeeringDB limits](https://docs.peeringdb.com/howto/work_within_peeringdbs_query_limits/)).
  Our sync: sequential requests, 3 s apart, at most once per 24 h unless `--force`. On 429, stop and
  keep the old data.
- **API key:** optional, from env `SPA_PEERINGDB_API_KEY`, sent as `Authorization: Api-Key <key>`
  ([API specs](https://docs.peeringdb.com/api_specs/)). Never logged, never written to disk by us.
- **Fields used** (UNVERIFIED, confirm against https://www.peeringdb.com/apidocs/ in M4):
  `ixpfx.prefix, ixpfx.protocol, ixpfx.ixlan_id`; `ixlan.id, ixlan.ix_id`;
  `ix.id, ix.name, ix.city, ix.country`; `netixlan.asn, ipaddr4, ipaddr6, ix_id, net_id`.
- **Store:** `datasets/peeringdb.sqlite` (our own normalized tables, not peeringdb-py's schema).
  Exact-IP dict for netixlan. Prefix match by scanning a sorted list of (network int, prefix length)
  per family. There are only a few thousand ixpfx rows, so a linear or bisect scan is fine.
- **Precedence:** an exact netixlan match sets `ixp.member_asn`, which overrides BGP origin in
  attribution. A prefix-only match marks "IXP fabric" with no member. A crossing is confirmed only
  with neighbor checks ([classifier.md §6](classifier.md#6-attribution-analyzeattributionpy)), following
  [traIXroute](https://labs.ripe.net/author/george_nomikos/detecting-ixps-in-traceroute-paths-using-traixroute/).
- **License/attribution:** PeeringDB data terms were not reviewed in the research. Credit PeeringDB
  in README and `spa data status`. Review the terms before any redistribution of the synced data (we never redistribute it).

### DB-IP Lite City: geolocation (low confidence)
- **What:** IP to City Lite, MMDB format. **License:** CC BY 4.0. Attribution to DB-IP is required
  ("IP Geolocation by DB-IP", link https://db-ip.com). Updated monthly
  ([DB-IP Lite](https://db-ip.com/db/lite.php)).
- **Download URL pattern and MMDB field names:** UNVERIFIED (the field list was not on the fetched
  page). Confirm in M4 and record them here.
- **Read with** `maxminddb` (memory-mapped, built-in longest-prefix match).
- **Confidence:** always `low` (`DB_CITY`). Free databases get about 78% of router IPs right at
  country level, and city-level results are weak
  ([Gharaibeh et al. via APNIC](https://blog.apnic.net/2017/11/03/trust-geolocation-databases-geolocate-routers/)).
  Rounded to 1 decimal of lat/lon. City is shown as `~City`.

### rDNS hostname hints
- PTR lookup with dnspython (timeout 2 s). A built-in table maps common IATA airport codes to city
  and coordinates (a small hand-maintained CSV in `enrich/data/iata.csv`, about 300 major airports; source
  for coordinates to be chosen and credited in M4). Match `(?<![a-z])([a-z]{3})\d*(?![a-z])` tokens
  in hostname labels against the table, and use a hint only if exactly one distinct code matches.
- Confidence `medium` (`RDNS_HINT`). Hostname conventions are operator-specific, and false matches happen.
- CAIDA Hoiho API is a possible later improvement ([CAIDA blog](https://www.caida.org/blog/hoiho-api-holistic-orthography-of-internet-hostname-observations/)).
  It is not used in v0.1 because terms and limits are unknown.

### Not used by default
- **MaxMind GeoLite:** needs a per-user account and license key, has a 30-day deletion rule and an
  attribution requirement ([EULA](https://www.maxmind.com/en/geolite2/eula)). Optional: if
  `SPA_MAXMIND_LICENSE_KEY` is set **and** the user put a GeoLite City MMDB path in config, use it in
  place of DB-IP. No automatic download in v0.1.
- **IPinfo Lite** (needs a token, CC BY-SA, country only), **RIPEstat**, **bgp.tools**, **CAIDA
  AS2Org/prefix2as:** documented alternatives only ([ADR-0004](../decisions/0004-ip-to-asn-source.md)).

## Cache (`store/cache.py`)

- **Location:** `platformdirs.user_cache_path("smart-path-analyzer")`, overridable with
  `--cache-dir` or `SPA_CACHE_DIR`. Datasets go in `<cache>/datasets/`, lookups in `<cache>/cache.sqlite`
  (WAL mode).
- **Schema:**
  ```sql
  CREATE TABLE ip_cache  (ip TEXT, source TEXT, payload TEXT, fetched_at INTEGER, ttl_s INTEGER,
                          PRIMARY KEY (ip, source));
  CREATE TABLE asn_cache (asn INTEGER, source TEXT, payload TEXT, fetched_at INTEGER, ttl_s INTEGER,
                          PRIMARY KEY (asn, source));
  CREATE TABLE datasets  (name TEXT PRIMARY KEY, version TEXT, etag TEXT, last_modified TEXT,
                          fetched_at INTEGER, path TEXT, license TEXT);
  ```
  `payload` is JSON. Expired rows are ignored and overwritten. There is no background cleanup.

| Entry | TTL | Basis |
|---|---|---|
| Team Cymru IP→ASN (hit) | 24 h | notes: "about 24h for IP→ASN from live APIs" |
| Team Cymru AS name | 7 days | notes: "7 days for AS names" |
| Negative (NXDOMAIN, timeout) | 1 h | notes: "negative-cache … about 1h" |
| rDNS PTR | 24 h | Suggested |
| iptoasn dataset | refresh after 24 h, warn after 7 days | Suggested |
| PeeringDB sync | refresh after 24 h, warn after 7 days | PeeringDB: sync "at least daily" |
| DB-IP Lite | refresh after 30 days, warn after 60 days | monthly upstream |

The time comes from an injectable `Clock` so tests are deterministic.

## HTTP downloads (`enrich/download.py`)

The stdlib `urllib.request` is used with a 60 s timeout. It follows redirects, streams to a temp
file, then `os.replace()` puts it in place atomically. The User-Agent is
`smart-path-analyzer/<version> (+<project URL>)`. Conditional requests use stored ETag and
Last-Modified. Gzip files are decompressed while converting.
<!-- Why: a handful of bulk downloads do not justify an HTTP client dependency, and httpx's maintenance status is uncertain (ADR-0004). -->

## Failure modes (degrade, never crash)

| Situation | Behavior | Shown as |
|---|---|---|
| No datasets yet (`spa data update` never run) | Team Cymru for ASNs; no IXP; no geo | Caveat `ENRICHMENT_DEGRADED` ("no local datasets") |
| `--offline` and no datasets | Hop numbers and IPs only; classifier still runs | Caveat `ENRICHMENT_DEGRADED` |
| Dataset older than its warn age | Use it | stderr warning plus `HopEnrichment.notes` |
| DNS timeout or SERVFAIL | Negative-cache 1 h, continue | `notes: "team-cymru timeout"` |
| PeeringDB 429 or 5xx during update | Keep old data, exit 4 from `spa data update` | message names the limit and the key env var |
| Corrupt dataset file | Ignore it, warn, suggest `spa data update --force` | stderr |
| Geo implausible by RTT | Keep it in JSON, mark `GEO_IMPLAUSIBLE`, exclude from the physics check | `~City?` in table |

Enrichment changes the location wording and the AS-boundary and unknown-AS confidence caps
([classifier.md §6](classifier.md#6-attribution-analyzeattributionpy)). It can change the verdict
**code** in exactly one case: a propagation-like latency step where medium-confidence (rDNS) geo
shows a `DETOUR` ([classifier.md §5.6](classifier.md#56-latency)). Loss verdicts never depend on enrichment.
