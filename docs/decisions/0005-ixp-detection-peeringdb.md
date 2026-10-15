---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0005: IXP detection from a daily PeeringDB bulk sync, with neighbor checks before claiming a crossing

## Context and Problem Statement

When a problem starts at an interconnection, the user benefits from knowing it happened at a named
IXP and which member network owns the interface. BGP origin lookups on IXP peering-LAN addresses
are often missing or return the IXP's own ASN. Free IXP data has strict rate limits, and an
address on a peering LAN alone does not prove that the path crossed the exchange.

## Decision Drivers

- Correct attribution at IXP hops (member ASN rather than IXP or no ASN).
- Stay well inside PeeringDB's limits.
- Avoid false "crossed IXP X" claims.

## Considered Options

1. **Bulk-fetch `ix`, `ixlan`, `ixpfx`, `netixlan` (depth=0) at most daily, look up locally (chosen)**
2. Query the PeeringDB API per hop
3. peeringdb-py library sync
4. CAIDA PeeringDB snapshots / PCH data

## Pros and Cons of the Options

### 1. Own bulk sync
- Good: about 4–5 requests per sync. Anonymous limits are 20/min, identical repeats 2/min (1/hour above 100 KB).
  The docs recommend ≥ 2 s spacing and a local cache synced daily
  ([PeeringDB limits](https://docs.peeringdb.com/howto/work_within_peeringdbs_query_limits/)).
- Good: no extra dependency. Our own small schema.
- Bad: field names are UNVERIFIED (the apidocs page did not render in the research), so confirm them in M4.

### 2. Per-hop API queries
- Bad: up to 30 queries per run, which breaks the anonymous limit and makes repeated identical queries likely.

### 3. peeringdb-py
- Good: official, incremental sync ([HOWTO](https://docs.peeringdb.com/howto/peeringdb-py/)).
- Bad: adds a dependency with its own config (`~/.peeringdb/config.yaml`). Its SQL schema "may change without
  notice". Whether an anonymous full sync works is unverified.

### 4. CAIDA snapshots / PCH
- Bad: request form under the CAIDA AUA ([CAIDA](https://www.caida.org/catalog/datasets/peeringdb/)). PCH was not researched.

## Decision Outcome

Chosen: **option 1**. `spa data update` fetches the objects sequentially, 3 s apart, at most once
per 24 h. It uses an optional `SPA_PEERINGDB_API_KEY` (header `Authorization: Api-Key <key>`; 40/min with a key,
[API specs](https://docs.peeringdb.com/api_specs/)) and an identifying User-Agent. On 429 it keeps the old data.

Lookup rules:
- An exact `netixlan` IP match gives the IXP and member ASN. The **member ASN overrides the BGP origin**
  for attribution.
- An `ixpfx` prefix-only match marks "IXP fabric", with no member.
- A crossing is claimed only if the previous public hop's ASN is a member of that IX, or the next
  hop's ASN equals the member ASN. Otherwise the report says "IXP address seen (crossing not confirmed)".
  This follows traIXroute: routers may reply from IXP-subnet addresses without the path crossing
  the IXP, and IXP prefix data can be wrong
  ([traIXroute](https://labs.ripe.net/author/george_nomikos/detecting-ixps-in-traceroute-paths-using-traixroute/);
  [arXiv 1611.03895](https://arxiv.org/abs/1611.03895)).

### Consequences

- Good: names IXPs and member networks without any per-run API traffic.
- Bad: up to 24 h stale. There is no PCH data, so IXPs missing from PeeringDB are invisible.
- PeeringDB data terms were not reviewed. We never redistribute synced data, and we credit PeeringDB.

### Confirmation

Unit tests with a fake opener and `FakeClock`: ≤ 5 requests, ≥ 2 s apart, key header only when the env var is set;
member ASN overrides origin; prefix-only gives no member; the crossing-confirmation rules cover both neighbor cases.

## More Information

[hop_enrichment: PeeringDB](../research.md#enrichment). Detection sketch: [enrichment.md](../spec/enrichment.md#peeringdb-ixp-detection).
