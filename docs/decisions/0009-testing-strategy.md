---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0009: Pure-function unit tests, Hypothesis properties, syrupy snapshots, replay fixtures, optional root-only netns lab

## Context and Problem Statement

The classifier has to be right about real network behavior, but CI has no root, no scamper and no
Internet probing. Agents need fast, deterministic checks they can run before declaring work done
("Give Claude a check it can run", [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)).

## Decision Drivers

- Deterministic CI without privileges.
- Coverage of statistical edge cases that hand-picked examples miss.
- Real packet behavior validated at least once per release.
- Output contracts protected against accidental change.

## Considered Options

1. **Layered: unit + Hypothesis + syrupy + replay in CI; netns/netem/iptables lab by hand (chosen)**
2. Live Internet tests in CI (Docker with NET_RAW)
3. containerlab or mininet with vendor router images
4. Example-based unit tests only

## Pros and Cons of the Options

### 1. Layered
- Good: the classifier is pure, so Hypothesis can generate thousands of physically consistent runs from a
  ground-truth builder (link loss propagates, ICMP drops do not), as the notes suggest.
- Good: syrupy snapshots of JSON, the Rich text and the schema catch contract drift ([syrupy](https://syrupy-project.github.io/syrupy/)).
- Good: the lab uses real kernels. `tc netem` gives real forwarding loss and delay
  ([tc-netem(8)](https://man7.org/linux/man-pages/man8/tc-netem.8.html)), and iptables `limit` on Time Exceeded
  gives cosmetic hop loss ([iptables-extensions(8)](https://ipset.netfilter.org/iptables-extensions.man.html)).
  iptables was validated as an emulator of router ICMP rate limiting by Guo & Heidemann ([PAM 2018](https://www3.isi.edu/~johnh/PAPERS/Guo18a.pdf)).
  `icmp_ratelimit=0` gives clean baselines ([ip-sysctl](https://docs.kernel.org/networking/ip-sysctl.html)).
- Bad: lab commands are unverified until M6. Lab runs depend on a human with root.

### 2. Live tests in CI
- Bad: flaky (real Internet), slow, and needs privileged runners.

### 3. containerlab / mininet
- Good: real vendor CoPP and MPLS behavior.
- Bad: heavy, vendor images, docs not researched. Possible later.

### 4. Examples only
- Bad: misses sampling-edge cases, which is exactly where statistical classifiers fail.

## Decision Outcome

Chosen: **option 1**. Layers, fixtures, properties P1–P12, snapshot policy, lab scripts and the
12 acceptance scenarios are specified in [testing.md](../testing.md). Lab tests are marked `@pytest.mark.lab`,
need root, and are excluded by default. Live tests are marked `@pytest.mark.live`. CI matrix: Python 3.12–3.14 on
Ubuntu plus 3.13 on macOS, using `astral-sh/setup-uv` with a pinned uv
([uv GitHub guide](https://docs.astral.sh/uv/guides/integration/github/)). Parser tests use only real recorded
scamper output, never hand-written JSON.

### Consequences

- Good: agents can run the full non-lab suite in seconds, with no privileges.
- Good: lab recordings become CI regression fixtures, so real behavior is replayed forever after.
- Bad: properties must be phrased to hold for every sample, or Hypothesis will find sampling flukes.

### Confirmation

`pyproject.toml` excludes `lab` and `live` by default. The CI workflow has no `sudo`. M6 acceptance
requires the pasted lab output.

## More Information

[loss_attribution Q6](../research.md#loss-attribution-and-statistics), [python_tooling §7](../research.md#python-tooling).
