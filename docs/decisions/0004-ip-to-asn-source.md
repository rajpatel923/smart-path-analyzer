---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0004: iptoasn.com offline dataset with bisect lookup; Team Cymru DNS as live fallback; stdlib HTTP

## Context and Problem Statement

Each hop needs origin ASN(s) and an AS name so verdicts can say "inside AS3356 (Lumen)". Lookups must
work offline, must not hit rate limits, must stay license-compatible with MIT, and should not need
compiled dependencies.

## Decision Drivers

- Offline-first reliability: free live APIs are throttled or have shut down before.
- Licensing friction for every end user (accounts, tokens, share-alike).
- Pure-Python install.
- Freshness of routing data.

## Considered Options

1. **iptoasn.com `ip2asn-combined.tsv.gz` + pure-Python bisect (chosen)**
2. IPinfo Lite (MMDB/CSV)
3. CAIDA RouteViews prefix2as + a radix trie (py-radix)
4. pyasn
5. Live only: Team Cymru DNS or whois, RIPEstat, bgp.tools

## Pros and Cons of the Options

### 1. iptoasn
- Good: public domain (ODC PDDL), updated hourly, v4 and v6, already includes `AS_description`.
  Non-overlapping ranges make bisect exact ([iptoasn.com](https://iptoasn.com/)).
- Good: no account. No compiled dependency (sorted ints + `bisect_right`).
- Bad: one origin ASN per range (no MOAS). Its public API was retired in 2020, so we depend on the file
  staying available. AS names are raw registry-style strings.

### 2. IPinfo Lite
- Good: cleaner `as_name`, `as_domain`, daily updates ([docs](https://ipinfo.io/developers/ipinfo-lite-database)).
- Bad: every user needs a token, and CC BY-SA 4.0 attribution applies ([ipinfo.io/lite](https://ipinfo.io/lite)).

### 3. prefix2as + py-radix
- Good: reproducible, daily, MOAS encoded ([CAIDA](https://www.caida.org/catalog/datasets/routeviews-prefix2as/)).
- Bad: no AS names. CAIDA AUA with a citation requirement. Adds a compiled dependency (wheels exist).

### 4. pyasn
- Bad: last release 2023-09-01, sdist only, tested up to 3.11, needs a compiler ([PyPI](https://pypi.org/project/pyasn/)).

### 5. Live APIs only
- Team Cymru DNS: free and cacheable, but no numeric limit is published, and heavy per-IP whois users get
  null-routed ([Team Cymru](https://www.team-cymru.com/ip-asn-mapping)). Fine as a fallback.
- RIPEstat: 8 concurrent requests per IP, registration requested above 1,000/day ([RIPEstat](https://stat.ripe.net/docs/data_api)).
- bgp.tools: needs a contact User-Agent, bans scrapers, data licensing unclear ([KB](https://bgp.tools/kb/api)).

### HTTP client for the downloads
- **stdlib `urllib.request` (chosen)**: only a handful of bulk GETs with conditional headers.
- httpx: no stable release since 0.28.1 (2024-12-06). Community reports of a maintenance freeze and forks
  ([PyPI](https://pypi.org/pypi/httpx/json); [HN, not primary](https://hn.nuxt.dev/item/49477212)).
- requests: stable but Apache-2.0 and sync-only. Not needed for this volume.

## Decision Outcome

Chosen: **option 1** as primary. **Team Cymru DNS** (`origin.asn.cymru.com`, `origin6.asn.cymru.com`,
`AS<n>.asn.cymru.com` via dnspython) is the live fallback for misses only. Its answers can contain
several origins, so `origin_asns` is a tuple. `spa data update` downloads with conditional requests,
converts once into SQLite, and lookups use bisect over sorted ranges. The cache is stdlib `sqlite3` under
`platformdirs.user_cache_path`, with TTLs from the research (24 h IP→ASN live, 7 d AS names, 1 h negative)
([enrichment.md](../spec/enrichment.md#cache-storecachepy)).

### Consequences

- Good: works offline after one download. No accounts. No compiled dependencies. Public-domain data, so
  no attribution obligation (we still credit it).
- Bad: AS names are less polished. MOAS shows only via the fallback. If the iptoasn file
  disappears, we need a new primary (IPinfo Lite or prefix2as), which would be a new ADR.
- The download URL is UNVERIFIED and gets confirmed in M3. The Team Cymru DNS TXT field order is
  confirmed with a recorded fixture.

### Confirmation

Unit tests on a tiny documentation-range TSV: first, last and between ranges, v6. A test asserts that no
HTTP client package appears in `[project.dependencies]`.

## More Information

[hop_enrichment research](../research.md#enrichment).
