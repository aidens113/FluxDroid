# FluxDroid: Codex specifics

Read AGENTS.md for project rules and validation commands. Install global MeritPrompts rules
with `node d:/MeritClaude/install.mjs --codex --apply` before starting the session.

Use the routing table in `d:/MeritClaude/CODEX.md`: GPT-6.1 Sol Max lead/heavy review, Sol Medium
standard workers, Sol High sub-supervisors, Luna High discovery/mechanical work, and Terra
Medium visual/writing work. Honor user selections and runtime availability. Do not duplicate
the model table here; keep project-specific overrides below. GPT-6.1 Sol Max is the ceiling;
never use Astra, including for fallbacks or review.

- Linux-only commands run through `wsl -d Ubuntu -- <cmd>` for local checks, or over SSH on the
  Linux host for AOSP builds and devices. WSL has no KVM, so never launch Cuttlefish there.
- Branch tool: `node d:/MeritClaude/tools/branch.mjs`. Working-docs audit:
  `node d:/MeritClaude/tools/working-docs-audit.mjs`.

This repository belongs to no lesson family. Read the global lessons in
`d:/MeritClaude/lessons/global/` when one of them is relevant.
