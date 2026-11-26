# Agent Instructions: FluxDroid

The global rules (roles, model routing, workflow modes, delegation, automatic git management)
load from MeritPrompts for every session. This file adds what is specific to this repository and
wins where the two conflict. Install global rules for the runtime in use first.
Codex sessions: read [CODEX.md](CODEX.md). Claude sessions: read [CLAUDE.md](CLAUDE.md).

## Purpose

FluxDroid is a family of virtual Android phones built on AOSP Cuttlefish. Each one presents a
coherent, realistic device identity and hardware profile instead of the stock "Cuttlefish"
identity. The repository holds the AOSP product definition and patches, device profiles, host
setup, the `fluxctl` instance manager, and the realism audit that measures how far a running
instance is from a real phone. It does not hold the AOSP source tree, built images, or the
Cuttlefish host packages. Those are fetched or built on the Linux host from pinned revisions.

## Scope Boundary

Realism means that identity and hardware observations are consistent and backed by working
features. It never means defeating cryptographic trust. Agents must not build, research, or
commit any of these, even when asked in a brief:

- Forging or replaying hardware-backed key attestation, or bypassing Play Integrity or
  SafetyNet remote verdicts.
- Reusing a certified retail device's build fingerprint, or any keys or credentials extracted
  from a physical device.
- Copying the real identifiers of a physical device or SIM (IMEI, MEID, serial, MAC, ICCID,
  IMSI, Android ID). Generated identifiers follow the policy in the plan.

If a task needs any of these to succeed, stop and report it as out of scope. Do not look for a
workaround.

## Start Here

Read the [handoff](docs/working/handoff.md) if it exists, then the
[working document index](docs/working/README.md) and the `Current State` of the relevant
document.

- [docs/working/fluxdroid-plan.md](docs/working/fluxdroid-plan.md): before planning or starting
  any phase work. It holds the phases, the host decision, and the realism-audit inventory.
- `docs/architecture/`: once it has content, read it when you change the build, profile, or
  instance contracts. Skip it for edits within one module.

Workers read only their brief.

## Two Machines

The development machine is Windows 10 with WSL2. WSL2 has **no `/dev/kvm`**, so Cuttlefish
cannot boot there and AOSP is not built there. Every build and boot runs on the **Linux host**
named in the plan's `Current State`. The Windows side may edit files, run `make check` inside WSL
(`wsl -d Ubuntu -- make check`), and drive the host over SSH. A brief that needs a running
device names the host and the instance.

## Repository Boundary

| Path | Owns |
| --- | --- |
| `manifests/` | `repo` local manifest and the pinned AOSP release snapshot |
| `device/fluxdroid/` | AOSP product tree (inherits a Cuttlefish target), overlays, sepolicy, props |
| `patches/<aosp-project>/` | Ordered patch series against AOSP projects, applied by `scripts/apply-patches.sh` |
| `profiles/` | Device profiles (JSON) and their schema: identity, resources, peripherals |
| `fluxctl/` | Python instance manager: create, start, stop, reset, install, logs, location |
| `audit/` | Realism audit: probes a running instance over adb and scores it against its profile |
| `host/` | Linux host provisioning scripts |
| `scripts/` | Build and sync entry points |

Dependencies go one way. `fluxctl` and `audit` read `profiles/`. `device/` and `patches/` never
import from Python code. `audit/` must not import from `fluxctl/`.

Never commit any of these:

- The AOSP tree, `out/`, images (`*.img`, `*.zip` image bundles), or `cvd-host_package.tar.gz`.
- Signing keys, AVB keys, or `releasekey`/`platform` key material. Keys live only on the host,
  in the path the plan names.
- Instance state (`cuttlefish_runtime*`, overlays, `~/cuttlefish*`) or adb keys.
- Captured data from real phones, and any account credentials.

Never let any of these appear in logs, reports, or responses: key material, adb private keys,
host SSH credentials, or account tokens.

## Code Structure

- Python 3.12 is used for `fluxctl/` and `audit/`, with `pyproject.toml` at the root. Lint and
  type checks use ruff and mypy (strict), and tests use pytest.
- Tests live in a `tests/` subfolder inside the directory that owns their subject, for example
  `audit/probes/tests/`.
- Every audit probe is a small module that returns findings and has a fixture-based test. No
  catch-all `utils.py` is allowed.
- Shell scripts are POSIX `bash` with `set -euo pipefail`, and must pass shellcheck.
- Each patch in `patches/<project>/` is a numbered `git format-patch` file with a message
  explaining which realism-audit check it closes.

## Validation

Run the narrowest relevant checks while iterating. Run these before the final response for a
code change, and pass them as `--check` to `branch.mjs finish` and `promote`:

```bash
wsl -d Ubuntu -- make check        # ruff, mypy, pytest, profile schema, shellcheck
node d:/MeritClaude/tools/working-docs-audit.mjs
```

Until Phase A lands the `Makefile`, only the working-docs audit exists.

Unit tests cannot prove that an image boots, that a prop or sysfs value changed on a running
device, or that a peripheral works. These need live validation on the host, and the report
states which instance, image build, and profile it exercised.

## Live Validation

- **The realism audit is the acceptance test for this project.** A change to identity or to the
  detection surface counts as done only when `audit` passes against a running instance of the
  target image. Record the audit JSON summary (score, failing checks) in the ledger.
- Before you patch a detection-surface item, add its audit check first and show it failing
  against the current image.
- Agents may create, start, stop, and reset instances whose names begin with `fd-test-`. Leave
  every other instance and its overlays alone.
- A full AOSP sync or build takes hours. State the target and expected time before you start
  one, and run it in the background with a watcher. Never start one unsupervised.
- Cloud hosts cost money. Starting, resizing, or leaving one running needs the user's approval
  for that run.
- Never sign into a real Google account or a real personal account on a test instance.

## Documentation Maintenance

After a substantial change, update the authored documentation in the same unit of work unless
the user says not to; when the user asks for documentation updates, they are required work.
Keep current-state design in `docs/architecture/` and task-specific plans in `docs/working/`.
These changes are substantial: the pinned AOSP release, the base Cuttlefish target, the profile
schema, the `fluxctl` command surface, the audit check inventory, the host layout, and the key
locations.
