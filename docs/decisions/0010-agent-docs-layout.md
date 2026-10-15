---
status: accepted
date: 2026-10-13
decision-makers: project lead
---

# ADR-0010: AGENTS.md as the single source of truth, a CLAUDE.md that imports it, and detail in docs/

## Context and Problem Statement

Claude Code and OpenAI Codex will both build this project. Each reads a different file by default, and
both are sensitive to instruction length and quality. We need one source of truth that both tools load
reliably, that stays short, and that points to the detailed specs.

## Decision Drivers

- Both tools must load the same rules.
- Works on all OS and tool versions.
- Short: Codex truncates the combined instruction files silently at 32 KiB by default
  ([Codex AGENTS.md docs](https://learn.chatgpt.com/docs/agent-configuration/agents-md);
  [Vaughan](https://codex.danielvaughan.com/2026/03/26/agents-md-advanced-patterns/)). Claude Code
  recommends keeping CLAUDE.md under 200 lines ([memory docs](https://code.claude.com/docs/en/memory)).
- Evidence on what helps. Developer-written files help modestly (about 4%). LLM-generated files slightly
  hurt. Overviews do not help. "Unnecessary requirements from context files make tasks harder"
  ([Gloaguen et al., ICLR 2026](https://arxiv.org/html/2602.11988v1)). A small 2026 ablation on Python
  repos found no clear effect either way ([Khatri](https://www.alphaxiv.org/abs/2607.27250)).

## Considered Options

1. **AGENTS.md canonical + CLAUDE.md containing `@AGENTS.md` and a few Claude-only lines (chosen)**
2. `CLAUDE.md -> AGENTS.md` symlink
3. AGENTS.md only, relying on Claude Code's native AGENTS.md reading
4. Two maintained copies
5. A long, all-in-one root file instead of `docs/`

## Pros and Cons of the Options

### 1. Import
- Good: officially documented. Works on every Claude Code version. "Never makes Claude read AGENTS.md
  twice" ([memory docs](https://code.claude.com/docs/en/memory)). Allows Claude-only additions.
- Good: a personal `CLAUDE.local.md` stays safe, because the import is explicit.
- Bad: one extra file.

### 2. Symlink
- Bad: on Windows without `core.symlinks`, Git checks it out as a one-line text file. Edit/Write tools refuse to
  write through symlinks ([memory docs](https://code.claude.com/docs/en/memory)).

### 3. AGENTS.md only
- Good: no extra file (Claude Code v2.1.277+ reads AGENTS.md natively).
- Bad: only when **no** CLAUDE.md or CLAUDE.local.md exists on the path. Adding a personal CLAUDE.local.md
  silently stops AGENTS.md from loading. Older versions and some configurations don't load it at all
  ([memory docs](https://code.claude.com/docs/en/memory)).

### 4. Copies
- Bad: drift. Contradictory instructions are resolved arbitrarily (memory docs). `/import` creates exactly this.

### 5. Long root file
- Bad: costs context every session, lowers adherence, and risks Codex truncation. Overviews are what the ETH study found useless.

## Decision Outcome

Chosen: **option 1**.
- `AGENTS.md` (hand-written, tool-neutral, aim ≤ 120 lines): purpose, exact commands, a planned repo map
  (kept because the repo starts empty and the map cannot be discovered yet; prune it once code exists),
  hard rules, workflow, gotchas, and a docs index of backticked paths, not `@` imports. Imports load at
  launch and don't reduce context cost, and Codex is assumed not to expand them.
- `CLAUDE.md`: `@AGENTS.md`, a maintainer HTML comment (Claude Code strips block comments before injection), and ≤ 10 Claude-only lines.
- Detail lives in `docs/`: specs with acceptance criteria, MADR 4.0 ADRs ([MADR](https://adr.github.io/madr/)), a roadmap with runnable checks.
- Rationale in AGENTS.md and the specs goes in short HTML comments pointing to ADRs. Claude Code strips them. Codex
  probably sees them, which costs a few bytes.
- Add a rule to AGENTS.md only after an agent makes the same mistake twice. Prefer enforcing with a test, lint or hook.
- `.gitignore` will list `CLAUDE.local.md` and `AGENTS.override.md` (personal overrides; Claude Code ignores the latter).

### Consequences

- Good: one place to edit. Both agents get the same rules and commands.
- Bad: Codex gets no automatic spec loading. It must open the files named in the workflow, as Claude does.
- The repo map in AGENTS.md is a deliberate exception to "no overview" until M5. Then it should shrink to the
  non-obvious parts (for example the purity rule for `analyze/`).

### Confirmation

`wc -c AGENTS.md` < 8 KiB. `head -1 CLAUDE.md` is `@AGENTS.md`. Checks to run: `/doctor prompt-audit` in Claude Code
(flags references to missing files and commands) and `codex "List the instruction sources you loaded."`.

## More Information

[agent_instruction_files research](../research.md#agent-instruction-files).
