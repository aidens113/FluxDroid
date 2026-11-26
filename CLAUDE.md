@AGENTS.md

# FluxDroid: Claude Code specifics

Start a task with `/resume` and end it with `/handoff`, then `/clear`. Everything durable belongs
in `AGENTS.md`, a lesson family, or MeritPrompts's global rules, not here.

- The Bash tool is Git Bash on Windows. Linux-only commands (`make`, `shellcheck`, `repo`,
  `cvd`) run through `wsl -d Ubuntu -- <cmd>` for local checks, or over SSH on the Linux host
  for builds and devices. Never try to launch Cuttlefish inside WSL, because there is no KVM.
- AOSP syncs and builds run for hours. Use `run_in_background` with a Monitor on the log. Do not
  poll in a loop.
- When you brief a worker that touches a running device, name the host, the instance
  (`fd-test-*`), and the image build id in the brief. Workers do not discover these themselves.
- Run the MeritPrompts tools as `node d:/MeritClaude/tools/<tool>.mjs`.
