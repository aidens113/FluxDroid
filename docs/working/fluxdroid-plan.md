# FluxDroid Plan

Status: Active
Status detail: Plan written. Phase A is ready to dispatch; Phase B onward waits on the user choosing the Linux host (Open Questions Q1).
Created: 2026-11-25
Last updated: 2026-11-25
Owner: Lead supervisor
Scope: From an empty repository to a realistic custom Cuttlefish phone (x86_64 for development, ARM64 as the target) with profiles, an instance manager, and a measured realism audit. Excludes attestation and integrity bypass (see AGENTS.md Scope Boundary).
Paired document: none
Related: [AGENTS.md](../../AGENTS.md), [protocol](./agent-working-doc-protocol.md)

---

## Current State

**True now.** The repository holds only the agent instructions and this plan. No code, no host,
no image. MeritPrompts is installed and current at `d:/MeritClaude`, and the `dev` branch
exists.

**Facts about this machine (probed 2026-11-25).** It has an i5-12400F (6 cores, 12 threads), 32
GB RAM, an RTX 3060, Windows 10 Home 22H2, a 512 GB NVMe for `C:` (27 GB free), and a 1 TB HDD
for `D:` (137 GB free). WSL2 Ubuntu has no `/dev/kvm` and no `vmx` flag, and Windows 10 offers
no supported nested virtualization in WSL2. **Cuttlefish cannot boot on this machine as
configured, and the free disk space is too small to build AOSP.**

**Recommended host (Q1).** Install Ubuntu 24.04 natively on this PC, on a new NVMe of 2 TB or at
least 1 TB, and dual-boot. That gives KVM, the RTX 3060 for gfxstream, and an x86_64 AOSP build
host at no hourly cost. ARM64 hardware is needed only from Phase F.

**Done:** nothing yet; see the Work Ledger.

**Not done:** phases A–G below.

**Next:** dispatch Phase A, which needs no Linux host, while the user settles Q1. Phase B
starts the day the host can boot stock Cuttlefish.

**Blocked:** Phases B–G are blocked on Q1 (the host).

**Schedule:** parallel lanes, measured from the day the host exists. Each step names its real
constraint.

| Step | Elapsed | Constraint |
| --- | --- | --- |
| Phase A | 2–3 days, starting now | Agent work only. Runs before the host exists. |
| Phase B | Day 1 on host | Host package install, and downloading stock images from ci.android.com |
| Phase C | Days 2–3 | `repo sync` (~150 GB) plus the first full build, about 4–8 h on 12 threads and 32 GB |
| Phases D and E | Days 4–14 | Rebuild-and-boot cycles. Incremental builds take 10–30 min, so D runs 3–4 lanes |
| x86_64 MVP | ~3 weeks after host | Every D check is green on the user build, and E's operations pass |
| Phase F | +1 week | ARM64 KVM host availability (Q2) |
| Phase G | Ongoing | Depends on the app list (Q3) |

---

## Objective

Build a virtual phone whose observable surface is coherent, as seen by an ordinary app (the
`untrusted_app` vantage point). It shows no "Cuttlefish", "virtio", "crosvm", or
"Android Virtual" strings. Its identity, CPU, display, sensors, battery, and telephony agree
with each other and with its profile, and every advertised capability works. "Realistic" is
measured, not asserted: the realism audit (Phase A) is the acceptance test for every phase.

Two facts shape the plan:

1. **x86_64 is the development target, not the realism target.** An x86_64 guest reports
   `Build.SUPPORTED_ABIS = x86_64`, which almost no retail phone does, and it cannot run
   arm64-only native libraries without proprietary translators (libhoudini and
   libndk_translation, which we do not use). The identity, tooling, and audit work transfer
   unchanged to ARM64 (Phase F).
2. **Anything an app can only learn through cryptographic attestation is out of reach.** This
   includes the Play Integrity device verdict and key attestation. The audit reports these as
   `out-of-scope`, never as failures to fix.

## Host Options

| Option | KVM | GPU | Builds AOSP | Cost | Role |
| --- | --- | --- | --- | --- | --- |
| H1: native Ubuntu 24.04 on this PC, new NVMe | Yes | RTX 3060 (gfxstream) | Yes (slowly at 32 GB; use `-j10` and swap) | One SSD | **Recommended** for build and x86_64 development (A–E) |
| H2: cloud x86 VM with nested virtualization | Yes (nested) | None unless a GPU VM | Yes | Hourly | Fallback if H1 is not possible |
| H3: ARM64 bare metal (Ampere or Graviton) | Yes | Usually none, so SwiftShader (a strong tell) | **No** (AOSP builds only on x86_64) | Monthly or hourly | Runtime for Phase F. Images are still built on H1 or H2 |

Verify before choosing H3: whether an ARM64 host with a usable GPU and bare-metal KVM is
available (for example, AWS Graviton with an NVIDIA GPU). Cuttlefish's ARM64 graphics
acceleration on that host is unproven until Phase F tests it.

## Architecture

- **Build-time identity** (one product per model family) lives in `device/fluxdroid/<model>/`,
  inheriting `aosp_cf_x86_64_phone` (later `aosp_cf_arm64_only_phone`). It sets `ro.product.*`,
  brand, fingerprint format, characteristics, overlays, the default app set, and sepolicy
  deltas. Deeper renames are made as patches to `device/google/cuttlefish` and other projects.
- **Launch-time configuration** (per instance) comes from the profile. `fluxctl` renders the
  profile into a `cvd` JSON config with CPUs, memory, display, storage, GPU mode, and camera
  settings, and into `androidboot.*` bootconfig values for per-instance identifiers.
- **Generated identifier policy.** Since Android 10, ordinary apps cannot read IMEI, serial, or
  the factory MAC. Wi-Fi MACs are randomized per network and locally administered on real phones
  too. So FluxDroid generates every identifier uniquely per instance from a seed stored with the
  instance, and never copies one from a physical device. Q4 settles the TAC and IMEI format.
- **Signing.** The project generates its own release keys and AVB keys on the host, outside
  the repository, so builds carry `release-keys` instead of `test-keys`. That fixes the
  self-reported strings only. Remote verdicts are unchanged and out of scope.
- **Audit vantage points.** The realism audit reads from two places:
  - **app**: a probe APK installed as an ordinary app. This is the acceptance vantage point.
  - **shell**: `adb shell`, used for diagnosis only. The shell can see things apps cannot.

  A check that only the shell can see is ranked lower.

## Detection Surface Inventory

Each row is one audit check. "Stock" is what we *expect* on unmodified Cuttlefish. Phase B
replaces every "expected" with the observed value from the baseline audit. "Layer" names where
the fix lives.

| ID | Surface (app vantage unless noted) | Stock (expected) | Layer | Phase |
| --- | --- | --- | --- | --- |
| ID-01 | `Build.MANUFACTURER/BRAND/MODEL/PRODUCT/DEVICE/BOARD` | Google, Android, "Cuttlefish x86_64 phone" | product | C |
| ID-02 | `Build.FINGERPRINT`, `TAGS`, `TYPE` | aosp_cf_…/userdebug/test-keys | product + keys | C |
| ID-03 | `Build.HARDWARE`, `ro.hardware`, `ro.boot.hardware` | `cutf_cvm` | patch (init, HAL lookup) | D2 |
| ID-04 | `ro.soc.manufacturer/model`, `Build.SOC_*` | empty or generic | product | D1 |
| ID-05 | Settings › About (device name, model) | Cuttlefish strings | overlay | C |
| ID-06 | Installed packages or components containing cuttlefish or emulator names | present | product + patch | D1 |
| ID-07 | `Build.SUPPORTED_ABIS` | x86_64 | target arch | F |
| ID-08 | `/proc/cpuinfo` model, implementer, part | host CPU model | kernel + host | D7/F |
| ID-09 | cpufreq or thermal sysfs presence (big.LITTLE frequencies) | absent | kernel + vendor | D7 |
| ID-10 | Sensor list: names, vendors, ranges, resolutions | generic or virtual names | sensors HAL patch | D3 |
| ID-11 | Sensor noise (zero variance at rest) | perfectly constant | sensors HAL patch | D3 |
| ID-12 | Battery level, charging state, temperature, voltage over time | static | health HAL patch | D4 |
| ID-13 | Operator name, MCC/MNC, network type | virtual operator | modem simulator patch | D5 |
| ID-14 | Wi-Fi SSID, BSSID, link details | virtual SSID | wifi or netsim config | D5 |
| ID-15 | Bluetooth adapter name and controller | virtual | product + patch | D5 |
| ID-16 | Input device names (`InputDevice.getName`) | virtio or virtual names | patch or crosvm config | D6 |
| ID-17 | Display xdpi/ydpi (real panels are non-round), refresh rates, display name | exact round DPI | profile + patch | D6 |
| ID-18 | GL_RENDERER, GL_VENDOR, Vulkan device name | host GPU via gfxstream, or SwiftShader | graphics (decision, Q5) | D8 |
| ID-19 | Camera IDs, characteristics, and advertised modes that actually work | emulated camera | camera config | D6/G |
| ID-20 | `ro.debuggable`, ADB enabled, developer options, SELinux mode | userdebug defaults | user build | C |
| ID-21 | `ro.boot.verifiedbootstate`, `ro.boot.flash.locked` | orange, unlocked | AVB config (self-reported only) | D1 |
| ID-22 | Kernel release (`os.version`), `/proc/version` | GKI with a build host string | kernel config | D7 |
| ID-23 | Memory and storage totals vs profile | match the VM | profile | E |
| ID-24 | Network interface names and IP ranges | virtual names and fixed subnets | patch + host netcfg | D5 |
| ID-25 | Shell-only: `/proc/cmdline`, `/sys/bus/virtio`, device tree, mounts | virtual strings | various | D, lower priority |
| ID-26 | Coherence: SoC, CPU count, cpuinfo, ABI, GPU, and RAM agree with the profile | n/a | audit rule | all |
| ID-X | Play Integrity verdict, key attestation chain | fails | **out of scope** | — |

## Phases

Each phase ends at a gate the lead checks itself.

**Phase A: Host-independent foundation (start now).** The lead first lands `pyproject.toml`,
the `Makefile` (`make check`), and the profile JSON schema as stable interfaces. Then four
lanes run in parallel:

- **A1 (profiles):** the profile schema and a default profile, `profiles/fd-one.json` (an
  invented brand, with 6 GB, 4 vCPUs, 1080×2400 at 420 DPI, 64 GB storage, and 60 Hz).
- **A2 (audit framework):** the adb transport, the probe interface, the scoring report (JSON
  and text), and shell probes for ID-01 to ID-06, ID-20 to ID-22, and ID-25, each with fixture
  tests.
- **A3 (probe APK):** a Gradle project built in WSL with the command-line SDK. It dumps the app
  vantage point of every ID row as JSON, and A2 parses that output.
- **A4 (fluxctl skeleton):** commands, profile loading, rendering the `cvd` JSON, and dry-run
  mode, with tests.

Gate: `make check` passes in WSL, and the audit scores a recorded stock-like fixture with every
expected failure flagged.

**Phase B: Host bring-up and stock baseline.** Run the `host/` provisioning script (Cuttlefish
host packages, kvm/cvdnetwork/render groups, NVIDIA driver). Boot the stock prebuilt
`aosp_cf_x86_64_phone` image from ci.android.com, with its host package from the same build.
Confirm gfxstream is active and the WebRTC display is reachable. Run the full audit and commit
`docs/architecture/baseline-audit.md`. Then update the inventory's "Stock" column to the
observed values. Gate: two `fd-test-*` instances boot at the same time, and the baseline is
recorded.

**Phase C: Reproducible build and first identity.** Run `repo init -b android-latest-release`
and snapshot the resolved manifest into `manifests/` with `repo manifest -r`. Add the local
manifest for `device/fluxdroid`. Build stock `aosp_cf_x86_64_phone-userdebug` once to prove the
host. Then build `fluxdroid_fd_one`, which inherits it and sets identity, overlays, and release
keys, in both `userdebug` and `user`. Gate: ID-01, ID-02, ID-05, and ID-20 pass on the user
build, from the app vantage point.

**Phase D: Surface hardening.** Each lane follows the same loop: write the check, show it red,
patch, rebuild, and show it green. Lanes that touch the same AOSP project are serialized,
because they would share one patch series.

| Lane | Rows | Writes |
| --- | --- | --- |
| D1 | ID-04, 06, 21 | `device/fluxdroid/` |
| D2 | ID-03 (the `ro.hardware` rename, the largest item) | `patches/device_google_cuttlefish/` |
| D3 | ID-10, 11 | sensors HAL patch |
| D4 | ID-12 | health HAL patch, after D2 because both touch `device/google/cuttlefish` |
| D5 | ID-13, 14, 15, 24 | modem simulator, wifi, and netsim patches |
| D6 | ID-16, 17, 19 | display and input patches |
| D7 | ID-08, 09, 22 | Research spike first. Can the guest kernel present a believable cpuinfo, cpufreq, and thermal surface without lying about behaviour? |
| D8 | ID-18 | Spike and Q5 decision only. No change before the user decides |

Gate: every in-scope row is green on the x86_64 user build except ID-07, which closes in
Phase F.

**Phase E: Instance manager (in parallel with D, touching no image files).** `fluxctl` gets
`create`, `start`, `stop`, `reset` (a clean overlay), `install`, `logs`, `location`, and
`audit`. It also gets per-instance generated identifiers and a state registry that ties
instance, image build, and profile together. Gate: two instances keep independent data across
restarts, a reset returns a clean device, and ID-23 passes.

**Phase F: ARM64 runtime.** Build the arm64-only variant of the product on the x86_64 build
host and boot it on the ARM64 KVM host (Q2). Re-run the full audit, then close ID-07 and the
ARM64 side of ID-08. Settle the GPU path on that host. Gate: the full audit is green on ARM64,
apart from the documented exceptions.

**Phase G: App compatibility.** Use the user's app list (Q3) in four workflows: a WebView app,
a native app, a camera or video-call app, and a location app. Repeat the open/close cycles. Each
failure is traced to a capability or a surface item and fed back into D or F.

## Work Ledger

### 2026-11-25 — Repository set up and plan written
- Agent: supervisor
- Changed: `AGENTS.md`, `CLAUDE.md`, `CODEX.md`, `README`, `docs/working/*`; `d:/MeritClaude` pulled 12 commits and reinstalled.
- Why: user asked for MeritPrompts setup, custom agent instructions, and a custom plan.
- Validation: `node d:/MeritClaude/install.mjs --apply` -> wrote agents, skills, settings; `wsl -d Ubuntu -- ls /dev/kvm` -> "No such file or directory"; working-docs audit result recorded at commit.
- Outcome: Accepted
- Follow-up: user answers Q1; dispatch Phase A.

## Open Questions

- **Q1 (user): Linux host.** Recommended: H1, native Ubuntu 24.04 on this PC with a new 1–2 TB
  NVMe. This is settled when the user confirms H1, or chooses H2 and approves its cost.
- **Q2 (user, by Phase F): ARM64 runtime host.** The decision needs which provider is used, and
  whether it has a GPU. It is settled by a priced option plus a Phase F spike.
- **Q3 (user, by Phase G): target apps.** Q3 is open until the user names the apps. The list
  decides whether Google Mobile Services matter. GMS is licensed separately from AOSP; the
  options are none or microG, and the user decides which.
- **Q4 (lead, Phase E): identifier formats.** This covers the IMEI TAC range and serial format.
  The rule is unique per instance, Luhn-valid, and never copied from a real device.
- **Q5 (user, after D8): GPU identity.** The choice is between reporting the real host GPU
  through gfxstream and presenting a mobile GPU name. Presenting a mobile name would contradict
  the GPU's actual behaviour, which affects coherence (ID-26), so D8 brings evidence first.
