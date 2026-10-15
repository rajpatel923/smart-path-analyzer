---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0008: Rich human report, versioned Pydantic-generated JSON, and verdict-based exit codes

## Context and Problem Statement

Two kinds of readers consume the result. People want a hop table and one sentence that says
where the problem is. Scripts want stable fields and a meaningful exit status. The output also has to
carry the evidence and caveats the classifier produces, and the attributions that licenses require.

## Decision Drivers

- One-sentence verdict up front, with evidence and caveats next to it.
- A machine contract that can evolve without silently breaking consumers.
- Scripts must be able to tell "network fine", "network broken", "can't tell" and "tool or user error" apart.
- `--json` output must be pipe-clean.

## Considered Options

### JSON contract
1. **Pydantic v2 models → committed JSON Schema (Draft 2020-12), `schema_version` field (chosen)**
   ([Pydantic JSON Schema](https://pydantic.dev/docs/validation/latest/concepts/json_schema/)).
2. Hand-written `dict`s: no schema, easy to drift.
3. msgspec: Beta classifier ([PyPI](https://pypi.org/pypi/msgspec/json)).

### Exit codes
1. **0 no problem / 1 problem / 2 inconclusive / 3 usage / 4 probing or environment (chosen)**.
2. 0 / non-zero only: scripts would have to parse JSON to tell inconclusive from broken.
3. Click defaults (usage = 2): collides with "inconclusive".

### Human output
1. **Rich table + verdict line + evidence + caveats + hints + data attribution (chosen)**. Rich 15 is stable
   and honors `NO_COLOR` ([PyPI](https://pypi.org/pypi/rich/json)).
2. Plain text: no alignment or color cues.

## Decision Outcome

Chosen: option 1 in each group. Details: [cli.md](../spec/cli.md#output) and
[data-model.md](../spec/data-model.md#json-output-schema-v1-spa-trace---json-spa-analyze---json).

| Code | Meaning |
|---|---|
| 0 | `NO_PROBLEM`, `ICMP_DEPRIORITIZATION_ONLY` |
| 1 | `LOSS_STARTS_AT`, `LATENCY_STARTS_AT` |
| 2 | `INSUFFICIENT_DATA`, `DESTINATION_UNREACHABLE`, `PATH_UNSTABLE` |
| 3 | usage or input error |
| 4 | probing, permission or environment error |
| 130 | second Ctrl-C |

Privileges: probing needs raw sockets. Supported: Linux (sudo or setuid scamper; `setcap` only if M0
confirms it), macOS (sudo), Docker with `--cap-add=NET_RAW --network host`. Windows is unsupported and
WSL2 is untested ([probing.md](../spec/probing.md#privileges-and-preflight)).

### Consequences

- Good: `spa trace x && echo ok` means "the network is fine". `$? == 2` means "measure longer or another way".
- Bad: Typer usage errors must be remapped from 2 to 3 (`standalone_mode=False`), which needs a test.
- Schema changes are deliberate: a snapshot test fails until the committed schema and version are updated.

### Confirmation

Property P8 (mapping is total, and the JSON `exit_code` equals the process code). The schema snapshot test.
`spa trace x --rounds abc` exits 3. Rich snapshots at width 120.

## More Information

[python_tooling: Pydantic, Rich, Typer](../research.md#python-tooling).
