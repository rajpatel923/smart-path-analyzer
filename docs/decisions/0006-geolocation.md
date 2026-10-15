---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0006: DB-IP Lite City + rDNS hints for location, always labeled low or medium confidence, with an RTT sanity check

## Context and Problem Statement

Location helps a user read a path ("Chicago to New York"), and it lets the classifier tell a
long-distance step from a real latency problem. Free geolocation is poor for router interfaces,
and some databases impose license burdens on every user.

## Decision Drivers

- Honesty: never present a router location as fact.
- License: OK for an open-source CLI; no per-user signup by default.
- Usefulness for the physics check in [classifier.md §5.6](../spec/classifier.md#56-latency).

## Considered Options

1. **DB-IP Lite City MMDB (CC BY 4.0) + rDNS IATA-code hints (chosen)**
2. MaxMind GeoLite City
3. IPinfo Lite
4. CAIDA Hoiho API for hostname-based geolocation
5. No geolocation

## Pros and Cons of the Options

### 1. DB-IP Lite + rDNS
- Good: CC BY 4.0 (attribution only), monthly updates, MMDB format, no account ([DB-IP Lite](https://db-ip.com/db/lite.php)).
  rDNS codes (IATA, CLLI, city names) are often the best router hint (hop_enrichment geo section).
- Bad: the Lite version is less accurate. Free databases geolocate routers correctly at country level only
  about 77.5–78.6% of the time, and city level is weak
  ([Gharaibeh et al. via APNIC](https://blog.apnic.net/2017/11/03/trust-geolocation-databases-geolocate-routers/)).
  rDNS conventions are operator-specific.

### 2. MaxMind GeoLite
- Good: well known; City, Country and ASN.
- Bad: needs an account and license key per user, a 30-day deletion rule, attribution, and limited
  third-party sharing ([EULA](https://www.maxmind.com/en/geolite2/eula); [updating](https://dev.maxmind.com/geoip/updating-databases)).
  Bundling is inappropriate.

### 3. IPinfo Lite
- Bad: country only, token required, CC BY-SA ([ipinfo.io/lite](https://ipinfo.io/lite)).

### 4. Hoiho API
- Good: learned per-domain hostname rules ([CAIDA blog](https://www.caida.org/blog/hoiho-api-holistic-orthography-of-internet-hostname-observations/)).
- Bad: no published terms, limits or accuracy. Another live dependency.

### 5. None
- Bad: loses the distance check and some readability.

## Decision Outcome

Chosen: **option 1**. It is read with `maxminddb`. Labels: rDNS hint means confidence `medium`; DB city
or country means `low`. City is shown as `~City`. Coordinates are rounded to 1 decimal. The registry
country from Team Cymru or iptoasn is never shown as a location. A pair of hops whose great-circle distance exceeds
100 km × (RTT_a + RTT_b) is physically impossible, so it gets marked `GEO_IMPLAUSIBLE` and excluded
from the physics check (fiber about 200 km/ms one way,
[Steenbergen](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf)). The physics check uses
only `medium` geo, which in practice means rDNS hints. MaxMind is optional: it is used only if the
user configures their own MMDB path. Attribution "IP Geolocation by DB-IP" (https://db-ip.com) appears in
every human report that used DB-IP data, in the JSON `data_sources`, in `spa data status` and in the README.

### Consequences

- Good: no accounts. Honest labels. Location can veto false "latency problem" verdicts on long-haul links.
- Bad: DB-IP city alone never enables the physics check, so many long-distance steps stay
  `PROPAGATION_STEP` (no verdict). We accept this, because abstaining is better than a wrong answer.
- The DB-IP download URL and MMDB field names are UNVERIFIED (M4). The test MMDB approach is decided in M4
  ([testing.md](../testing.md#fixtures)).

### Confirmation

Tests: the IATA parser; the implausibility rule; DB-IP results never get `medium`; the attribution string
is present in a Rich snapshot whenever DB-IP data was used.

## More Information

[hop_enrichment: geolocation](../research.md#enrichment).
