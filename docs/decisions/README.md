# Architecture Decision Records

The format is [MADR 4.0](https://adr.github.io/madr/): YAML front matter (`status`, `date`,
`decision-makers`), then Context and Problem Statement, Decision Drivers, Considered Options, Pros and
Cons, Decision Outcome (with Consequences and Confirmation), and More Information.

Rules:
- Number ADRs sequentially (`NNNN-title-with-dashes.md`) and never renumber them.
- An accepted ADR is not edited for substance. To change a decision, write a new ADR with
  `status: accepted` that says `Supersedes ADR-NNNN`, and set the old one's status to `superseded by ADR-MMMM`.
- Adding a runtime dependency, changing a threshold default, the verdict enum, exit codes or the JSON
  schema major version requires an ADR.
- Every option must cite the research or a fetched source ([research.md](../research.md)).

| ADR | Title | Status |
|---|---|---|
| [0001](0001-language-and-tooling.md) | Python 3.12+, uv, Typer + Rich, dataclasses + Pydantic at the boundary, Ruff, mypy strict | accepted |
| [0002](0002-probing-backend.md) | scamper as a subprocess with JSON output is the probing backend | accepted |
| [0003](0003-measurement-strategy.md) | N repeated single-attempt Paris rounds on one fixed flow, about 1 per second, default 100 | accepted |
| [0004](0004-ip-to-asn-source.md) | iptoasn.com offline dataset with bisect lookup; Team Cymru DNS as live fallback; stdlib HTTP | accepted |
| [0005](0005-ixp-detection-peeringdb.md) | IXP detection from a daily PeeringDB bulk sync, with neighbor checks | accepted |
| [0006](0006-geolocation.md) | DB-IP Lite City + rDNS hints, labeled by confidence, with an RTT sanity check | accepted |
| [0007](0007-classifier-statistics-and-thresholds.md) | Persistence-based classifier with Wilson/Newcombe statistics and named thresholds | accepted |
| [0008](0008-output-and-exit-codes.md) | Rich human report, versioned Pydantic JSON, verdict-based exit codes | accepted |
| [0009](0009-testing-strategy.md) | Unit + Hypothesis + syrupy + replay in CI; root-only netns lab | accepted |
| [0010](0010-agent-docs-layout.md) | AGENTS.md as the source of truth, CLAUDE.md imports it, detail in docs/ | accepted |
