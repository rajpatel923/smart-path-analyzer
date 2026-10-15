@AGENTS.md

<!--
Why an @import instead of a symlink or a copy (see docs/decisions/0010-agent-docs-layout.md):
- When a CLAUDE.md exists, Claude Code reads AGENTS.md only if CLAUDE.md imports it. Native AGENTS.md
  reading (v2.1.277+) happens only when NO CLAUDE.md/CLAUDE.local.md exists on the path.
  https://code.claude.com/docs/en/memory
- A `CLAUDE.md -> AGENTS.md` symlink breaks on Windows clones without core.symlinks. The clone then
  gets a one-line text file instead of the instructions. Edit/Write tools also refuse to write
  through symlinks.
- A copy drifts, and contradictory instructions get resolved arbitrarily.
- A personal CLAUDE.local.md is safe here. In an AGENTS.md-only repo it would silently stop
  AGENTS.md from loading, but this CLAUDE.md imports AGENTS.md explicitly.
Edit AGENTS.md, not this file, for anything that both Claude Code and Codex need.
-->

## Claude Code only

- Use plan mode for any change that touches more than one module. Skip it if you can describe the
  diff in one sentence.
- When compacting, keep the list of modified files, the current milestone and the four check
  commands.
- Mention docs by backticked path. Do not `@`-import them, because imports load at launch and cost
  context every session.
