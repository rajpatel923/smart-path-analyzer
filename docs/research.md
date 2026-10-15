# Research sources

This file collects the sources behind the design. All of them were gathered on 2026-10-13 for five
research notes: probing engine, hop enrichment, loss attribution, agent instruction files and
Python tooling. Facts marked "PyPI JSON" were read from `https://pypi.org/pypi/<pkg>/json` on that date.

## Probing and Paris traceroute

- [Augustin et al., Detection, Understanding, and Prevention of Traceroute Measurement Artifacts (arXiv:0904.2733)](https://arxiv.org/pdf/0904.2733): fields that load balancers hash; how Paris keeps them constant for UDP, ICMP and TCP; how much artifact reduction it achieves.
- [Vermeulen et al., Multilevel MDA-Lite Paris Traceroute (IMC 2018)](https://ar5iv.arxiv.org/html/1809.10070): MDA cost; RIPE Atlas uses single-flow Paris.
- [M-Lab, Extracting Single-Path Traceroutes (2022)](https://www.measurementlab.net/blog/2022-extracting-single-traceroute/): scamper `tracelb` JSON line structure.
- [CAIDA scamper page](https://www.caida.org/catalog/software/scamper/): release cvs-20260902, GPLv2, output formats, install paths.
- [scamper(1) man page](https://www.caida.org/catalog/software/scamper/man/scamper.1.pdf): `trace -P` methods (UDP-Paris default), `-q`, `-w`, `-W`, privileges and privsep, control sockets.
- [scamper Python module docs](https://www.caida.org/catalog/software/scamper/python/) and [source](https://www.caida.org/catalog/software/scamper/python/_sources/index.rst.txt): `do_trace` arguments, `ScamperTraceHop` attributes. We don't use the module, but the attribute names hint at the JSON fields.
- [PyPI scamper](https://pypi.org/project/scamper/), [Homebrew formula](https://formulae.brew.sh/formula/scamper), [SUSE PackageHub](https://packagehub.suse.com/packages/scamper/20250227-bp160_1_10/)
- [Trippy CLI reference](https://trippy.rs/reference/cli/), [Trippy privileges](https://trippy.rs/guides/privileges/), [trippy RELEASES.md](https://raw.githubusercontent.com/fujiapple852/trippy/refs/heads/master/RELEASES.md), [crates.io trippy](https://crates.io/crates/trippy), [trippy GitHub](https://github.com/fujiapple852/trippy)
- [mtr man page source](https://github.com/traviscross/mtr/blob/master/man/mtr.8.in), [mtr ui/mtr.c](https://github.com/traviscross/mtr/blob/master/ui/mtr.c), [mtr(8) Debian](https://manpages.debian.org/testing/mtr/mtr.8.en.html): no Paris mode; report default of 10 cycles.
- [icmplib GitHub](https://github.com/ValentinBELYN/icmplib), [icmplib unprivileged doc](https://github.com/ValentinBELYN/icmplib/blob/main/docs/6-use-icmplib-without-privileges.md)
- [dublin-traceroute GitHub](https://github.com/insomniacslk/dublin-traceroute), [releases](https://github.com/insomniacslk/dublin-traceroute/releases); [libparistraceroute](https://github.com/libparistraceroute/libparistraceroute); [Debian paris-traceroute](https://dyn.manpages.debian.org/buster/paris-traceroute/paris-traceroute.1)
- [Scapy usage](https://scapy.readthedocs.io/en/latest/usage.html), [Scapy troubleshooting](https://scapy.readthedocs.io/en/latest/troubleshooting.html), [PyPI scapy](https://pypi.org/project/scapy/), [Scapy README](https://raw.githubusercontent.com/secdev/scapy/master/README.md)
- [icmp(7)](https://man7.org/linux/man-pages/man7/icmp.7.html), [Linux ip-sysctl](https://docs.kernel.org/networking/ip-sysctl.html): `ping_group_range`, `icmp_ratelimit`, `icmp_ratemask`.
- [Datadog CIS Docker 5.3](https://docs.datadoghq.com/security/default_rules/cis-docker-1.2.0-5.3), [AlmaLinux skopeo commit](https://git.almalinux.org/rpms/skopeo/commit/5109d70d85d3b5e1e4d024657ab2064c0c76620c): NET_RAW is a Docker default but was removed from Podman's default list.
- [microsoft/WSL#10731](https://github.com/microsoft/WSL/issues/10731)
- [IETF 99 MAPRG: Rate-limiting of IPv6 traceroutes is widespread](https://datatracker.ietf.org/meeting/99/materials/slides-99-maprg-rate-limiting-of-ipv6-traceroutes-is-widespread-measurements-and-mitigations-02.pdf)

## Loss attribution and statistics

- [Steenbergen, A Practical Guide to (Correctly) Troubleshooting with Traceroute (NANOG 47, book format)](https://major.io/wp-content/uploads/2012/06/RAS_Traceroute_Book_Format.pdf): slow-path ICMP, the persistence rule, asymmetry, MPLS repeating latency, fiber speed, interconnect /30s.
- [Cloudflare Learning: What is MTR](https://www.cloudflare.com/learning/network-layer/what-is-mtr/): CoPP, rate-limited hops with 0% destination loss, loss on the return path.
- [APNIC Blog (Lavin), How to properly interpret a traceroute or MTR](https://blog.apnic.net/2022/03/28/how-to-properly-interpret-a-traceroute-or-mtr/): "unless loss … is seen on every hop … it is not a problem".
- [RFC 4443](https://www.rfc-editor.org/rfc/rfc4443.txt) (ICMPv6 rate limiting), [RFC 4950](https://www.rfc-editor.org/rfc/rfc4950.txt) (MPLS ICMP extensions), [RFC 3443](https://www.rfc-editor.org/rfc/rfc3443.txt) (MPLS TTL models)
- [Guo & Heidemann, Detecting ICMP Rate Limiting in the Internet (PAM 2018)](https://www3.isi.edu/~johnh/PAPERS/Guo18a.pdf): token-bucket regimes, on/off alternation, iptables as an emulator, "cannot tell" outcomes.
- Ravaioli et al., Characterizing ICMP Rate Limitation on Routers: [record](https://science.ulysseus.eu/records/sj2c5-0r686), [HAL](https://hal.archives-ouvertes.fr/hal-01111190)
- [Cisco IOS ICMP Unreachable Rate Limiting](https://www3-realm.cisco.com/en/US/docs/ios/12_4t/12_4t2/hticmpun.pdf), [CIS Juniper benchmark item](https://www.tenable.com/audits/items/CIS_Juniper_OS_v2.1.0_L2.audit:0bfa3f2c9f9facee5e282f6226d5e2ff)
- [Fontugne et al., Pinpointing Delay and Forwarding Anomalies (arXiv 1605.04784)](https://arxiv.org/pdf/1605.04784), [RIPE Labs article](https://labs.ripe.net/Members/romain_fontugne/pinpointing-delay-and-forwarding-anomalies-in-ripe-atlas-built-in-measurements): differential RTT, median with a Wilson order-statistic CI, asymmetry.
- [Donnet et al., Revealing MPLS Tunnels Obscured from Traceroute](https://www.caida.org/catalog/papers/2012_revealing_mpls_tunnels/revealing_mpls_tunnels.pdf), [CAIDA page](https://www-old.caida.org/publications/papers/2012/revealing_mpls_tunnels/)
- [Viger/Augustin et al. (arXiv 0904.2733)](https://arxiv.org/abs/0904.2733), [HAL: Avoiding traceroute anomalies with Paris traceroute](https://hal.archives-ouvertes.fr/hal-01097553)
- [Luckie et al., bdrmap](https://users.caida.org/~mjl/pubs/bdrmap.pdf): why simple IP-to-AS mapping puts AS borders in the wrong place.
- [NIST/SEMATECH e-Handbook 7.2.4.1](https://itl.nist.gov/div898/handbook/prc/section2/prc241.htm): Wilson score interval.
- [tc-netem(8)](https://man7.org/linux/man-pages/man8/tc-netem.8.html), [iptables-extensions(8)](https://ipset.netfilter.org/iptables-extensions.man.html)

## Enrichment

- [Team Cymru IP to ASN Mapping](https://www.team-cymru.com/ip-asn-mapping)
- [iptoasn.com](https://iptoasn.com/)
- [RIPEstat Data API](https://stat.ripe.net/docs/data_api), [network-info](https://stat.ripe.net/docs/data-api/api-endpoints/network-info), [prefix-overview](https://stat.ripe.net/docs/data-api/api-endpoints/prefix-overview), [as-overview](https://stat.ripe.net/docs/data-api/api-endpoints/as-overview)
- [pyasn GitHub](https://github.com/hadiasghari/pyasn), [pyasn PyPI](https://pypi.org/project/pyasn/), [py-radix PyPI](https://pypi.org/project/py-radix), [pytricia GitHub](https://github.com/jsommers/pytricia), [pytricia PyPI](https://pypi.org/project/pytricia/)
- [CAIDA prefix2as](https://www.caida.org/catalog/datasets/routeviews-prefix2as/), [CAIDA AS Organizations](https://www.caida.org/catalog/datasets/as-organizations/), [CAIDA PeeringDB snapshots](https://www.caida.org/catalog/datasets/peeringdb/)
- [bgp.tools API KB](https://bgp.tools/kb/api), [bgp.tools pricing](https://bgp.tools/pricing)
- [IPinfo Lite DB docs](https://ipinfo.io/developers/ipinfo-lite-database), [ipinfo.io/lite](https://ipinfo.io/lite), [IPinfo Lite FAQ](https://community.ipinfo.io/t/ipinfo-lite-faq/6768)
- [MaxMind GeoLite](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data/), [GeoLite EULA](https://www.maxmind.com/en/geolite2/eula), [MaxMind updating databases](https://dev.maxmind.com/geoip/updating-databases)
- [DB-IP Lite](https://db-ip.com/db/lite.php)
- [PeeringDB API specs](https://docs.peeringdb.com/api_specs/), [Work within query limits](https://docs.peeringdb.com/howto/work_within_peeringdbs_query_limits/), [Web rate limiting blog](https://docs.peeringdb.com/blog/web_rate_limiting/), [Update Oct 2025–Apr 2026](https://docs.peeringdb.com/blog/update_oct_2025-apr_2026/), [peeringdb-py HOWTO](https://docs.peeringdb.com/howto/peeringdb-py/), [peeringdb-py GitHub](https://github.com/peeringdb/peeringdb-py)
- [RIPE Labs: traIXroute](https://labs.ripe.net/author/george_nomikos/detecting-ixps-in-traceroute-paths-using-traixroute/), [arXiv 1611.03895](https://arxiv.org/abs/1611.03895)
- [APNIC blog: router geolocation accuracy](https://blog.apnic.net/2017/11/03/trust-geolocation-databases-geolocate-routers/), [CAIDA paper PDF](https://www.caida.org/publications/papers/2017/look_at_router_geolocation/look_at_router_geolocation.pdf)
- [CAIDA Hoiho API blog](https://www.caida.org/blog/hoiho-api-holistic-orthography-of-internet-hostname-observations/), [sc_hoiho man page](https://www.caida.org/catalog/software/scamper/man/sc_hoiho.1.pdf)
- [IANA IPv4 special registry](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml), [IANA IPv6 special registry](https://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xhtml), [Python ipaddress](https://docs.python.org/3/library/ipaddress.html)

## Python tooling

- [Python devguide: versions](https://devguide.python.org/versions/): 3.10 reached end of life on 2026-10-01; 3.12 reaches it in 2028-10.
- uv: [Creating projects](https://docs.astral.sh/uv/concepts/projects/init/), [Locking and syncing](https://docs.astral.sh/uv/concepts/projects/sync/), [GitHub Actions](https://docs.astral.sh/uv/guides/integration/github/), [pre-commit](https://docs.astral.sh/uv/guides/integration/pre-commit/)
- [Typer release notes](https://typer.tiangolo.com/release-notes/), [Cyclopts vs Typer](https://cyclopts.readthedocs.io/en/latest/vs_typer/README.html)
- [Pydantic JSON Schema](https://pydantic.dev/docs/validation/latest/concepts/json_schema/)
- [Ruff formatter](https://docs.astral.sh/ruff/formatter/), [Astral blog: ty](https://astral.sh/blog/ty)
- [syrupy docs](https://syrupy-project.github.io/syrupy/)
- [Python asyncio subprocesses](https://docs.python.org/3/library/asyncio-subprocess.html)
- httpx situation: [PyPI httpx](https://pypi.org/pypi/httpx/json), [PyPI httpx2](https://pypi.org/pypi/httpx2/json), [HN thread (community, not primary)](https://hn.nuxt.dev/item/49477212)
- Licensing: [GNU GPLv2 FAQ (mirror)](https://www.southampton.ac.uk/~qs2g22/mirror/www.gnu.org/licenses/old-licenses/gpl-2.0-faq.html), [Apache: GPL compatibility](https://www.apache.org/licenses/GPL-compatibility.html)
- PyPI JSON for versions and licenses: [uv](https://pypi.org/pypi/uv/json), [ruff](https://pypi.org/pypi/ruff/json), [ty](https://pypi.org/pypi/ty/json), [mypy](https://pypi.org/pypi/mypy/json), [typer](https://pypi.org/pypi/typer/json), [rich](https://pypi.org/pypi/rich/json), [pydantic](https://pypi.org/pypi/pydantic/json), [pytest](https://pypi.org/pypi/pytest/json), [hypothesis](https://pypi.org/pypi/hypothesis/json), [syrupy](https://pypi.org/pypi/syrupy/json), [pytest-cov](https://pypi.org/pypi/pytest-cov/json), [platformdirs](https://pypi.org/pypi/platformdirs/json), [diskcache](https://pypi.org/pypi/diskcache/json), [scapy](https://pypi.org/pypi/scapy/json)

## Agent instruction files

- [Claude Code: How Claude remembers your project](https://code.claude.com/docs/en/memory), [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices), [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [OpenAI Codex: Custom instructions with AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md), [Codex: Build skills](https://learn.chatgpt.com/docs/build-skills.md)
- [agents.md](https://agents.md/)
- [Gloaguen et al., Evaluating AGENTS.md (arXiv 2602.11988, ICLR 2026)](https://arxiv.org/abs/2602.11988v1); [Khatri, Two-Agent Ablation (arXiv 2607.27250)](https://www.alphaxiv.org/abs/2607.27250)
- [HumanLayer, Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md), [Upsun, your AGENTS.md is probably too long](https://developer.upsun.com/posts/ai/agents-md-less-is-more), [Daniel Vaughan, AGENTS.md Advanced Patterns](https://codex.danielvaughan.com/2026/03/26/agents-md-advanced-patterns/)
- [adr.github.io](https://adr.github.io/), [MADR](https://adr.github.io/madr/), [Nygard, Documenting Architecture Decisions](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions)

## Known unknowns / verify before relying

These come from the "Gaps" sections of the research notes. Items marked **M0** block parser work.

### Probing (scamper)
1. **M0:** The exact scamper CLI to run one `trace` with JSON output to stdout, and flags for sport, dport, first and max TTL and gap limit. Only `-P`, `-q`, `-w` and `-W` are confirmed.
2. **M0:** JSON field names, RTT encoding, line structure for `trace`, whether unanswered probes appear, and how to get `probed_ttls`.
3. **M0:** Whether separate scamper processes honor the same fixed sport. How the ICMP-Paris checksum is pinned and reported.
4. **M0:** Whether `setcap cap_net_raw+ep` works for scamper (privsep/chroot), the unprivileged error output, and Debian/Ubuntu package names (the PPA could not be fetched).
5. **M0:** RFC 4950 MPLS label representation in scamper JSON.
6. WSL2: whether Time Exceeded comes back through NAT or mirrored mode (no source found).
7. trippy's JSON schema (stretch goal), and whether trippy has been active since its 2025-05-05 release.
8. MDA stopping-rule values: M-Lab's "six flows" conflicts with MDA-Lite's n₁=9 (stretch goal only).

### Enrichment
9. The iptoasn download URL path and DB-IP Lite download URL and MMDB field names (M3/M4).
10. PeeringDB `ixpfx`, `ixlan`, `ix` and `netixlan` field names. The apidocs page did not render. Verify at https://www.peeringdb.com/apidocs/ (M4).
11. Whether a full anonymous PeeringDB fetch of these objects works without a key, and whether limits on identical repeats affect large list responses (M4).
12. The field order of Team Cymru DNS TXT answers (whois verbose order is documented; DNS is assumed similar). There is no numeric rate limit.
13. PeeringDB data-use terms, RIPEstat terms and bgp.tools data licensing were not reviewed.
14. Router geolocation accuracy figures date from 2017 (IPv4 only). There are no Hoiho API terms or accuracy figures.
15. Whether Python's 3.13 `is_private` changes were backported to 3.12.x. This is moot because we use our own table.

### Classifier and lab
16. No published threshold exists for a "significant latency step" from a single vantage point. Our 10 ms floor is a suggestion.
17. The route-inflation factor of 1.5–2× over great-circle distance is unsourced, and our default of 2.0 is a suggestion.
18. No per-vendor default table for Time Exceeded generation rates exists. Steenbergen's 50–300 pps is the only general figure.
19. Whether locally generated ICMP errors go through the iptables OUTPUT chain (inference). There is no known tc recipe that delays only ICMP type 11 (lab scenario 6). nftables equivalents are unverified.
20. IPv6 `icmp/ratelimit` default (the sysctl page fetch was truncated).
21. RFC 1812 §4.3.2.8 could not be fetched. The Ravaioli et al. full text is unavailable (its 60% figure is secondhand).
22. How often asymmetric return paths cause wrong attribution from a single vantage point. Unquantified, hence the permanent `RETURN_PATH` caveat.

### Tooling
23. How the Typer 0.27 usage-error exception maps to exit codes, since Typer has vendored Click since 0.26. Pin it with a test.
24. ty has no stable release date. mypy 2.x config changes since 1.x were not checked.
25. Exact `ruff-pre-commit` hook ids and rev. Whether `pydantic.mypy` works with mypy 2.4.
26. Whether stdlib dataclasses work in `TypeAdapter` (believed yes, not verified). We don't rely on it: JSON models are explicit BaseModels.
27. Whether Codex expands `@path` imports in AGENTS.md (assume not; AGENTS.md uses plain backticked paths).
