# cachyos-tuning-audit

Version-pinned deep-research prompt for auditing `ry-install.fish` against current
upstream sources. Each release of the prompt tracks one exact script version; the
current file targets **v7.77.1**.

- **File:** `cachyos-tuning-audit.md`
- **Prompt version:** v7.77.1 (matches script version under audit)
- **Source of truth:** script > README > CHANGELOG

## Purpose

Drives an item-by-item, evidence-backed tuning review of a CachyOS gaming/LLM desktop
profile on a single, fixed hardware target. The prompt instructs the reviewer to
evaluate every config decision against current upstream documentation, flag anything
superseded or harmful, quantify safety deltas, respect deliberate
performance-over-safety trade-offs without auto-reverting them, and — new in this
revision — assess gaming-relevant knobs the profile does **not** yet set.

The deliverable is a **recommendations report**, not a modified script.

## Target hardware

Beelink GTR9 Pro — Ryzen AI Max+ 395 "Strix Halo" (Zen 5, 16C/32T, gfx1151) ·
Radeon 8060S (40 RDNA 3.5 CUs) · XDNA 2 NPU · 128 GB LPDDR5X-8000 unified (≤96 GB as
VRAM) · dual M.2 NVMe (ext4) · dual 10 GbE (RTL8127) + Wi-Fi 7 (MT7925) + BT 5.4 ·
140 W TDP · CachyOS · systemd-boot.

## Profile counts

From `_ir_validate_counts` (all 19 hard-asserted):

```
║ GLOBAL              ║ COUNT ║
║─────────────────────║───────║
║ KERNEL_PARAMS       ║ 16    ║
║ MKINITCPIO_HOOKS    ║ 11    ║
║ MKINITCPIO_MODULES  ║ 1     ║
║ LOGIND_IGNORE_KEYS  ║ 8     ║
║ ENV_VARS            ║ 11    ║
║ SYSCTL_VALUES       ║ 9     ║
║ PKGS_ADD            ║ 17    ║
║ PKGS_DEL            ║ 9     ║
║ MASK                ║ 10    ║
║ EXPECTED_VULKAN_PKGS║ 2     ║
║ EXPECTED_SERVICES   ║ 5     ║
║ _RY_POST_HOOKS      ║ 18    ║
║ _RY_BOOT_CRITICAL   ║ 4     ║
║ _RY_PHASE_NAMES     ║ 6     ║
║ _RY_BACKUP_TARGETS  ║ 2     ║
║ _RY_TMPDIR_GLOBS    ║ 6     ║
║ SYSTEM_DESTINATIONS ║ 15    ║
║ USER_DESTINATIONS   ║ 3     ║
║ managed files       ║ 18    ║  (15 system + 3 user)
║ MangoHud directives ║ 17    ║  (+ 1 commented: cpu_temp)
║ NTSYNC autoload conf║ 0     ║  (de-managed; assert-only)
```

Hard floors: **KERNEL_MIN 6.18** (preflight hard-fail) · CPU gate `Ryzen AI Max` ·
soft Mesa < 26.0 warn · linux-firmware advisory (hard-warn `20251125*` MES blob,
soft-warn < `20260110`; non-fatal).

## What changed since v7.75.1

The prompt's leading "What changed since v7.75.1" section enumerates the deltas the
reviewer must re-evaluate with fresh evidence. Headline items:

- **IOMMU INVERTED a second time: `iommu=pt` → `amd_iommu=off`** (v7.77.0) — AMD-Vi is
  now fully disabled (no PCI passthrough). KERNEL_PARAMS stays 16 (one-for-one swap).
  Every prior "IOMMU restored / AMD-Vi on / closed reduction" statement is now wrong;
  it is now the #2 **open** security reduction. README ships a VFIO/SR-IOV escape hatch
  (`amd_iommu=on iommu=pt`).
- **New verify sub `_vrkm_iommu`** (v7.77.0) — derives expected IOMMU state from
  KERNEL_PARAMS and asserts it against live `/sys/kernel/iommu_groups` + dmesg; with
  `amd_iommu=off` it hard-fails if any groups are present.
- **ntsync DE-MANAGED** (v7.76.1) — the `/etc/modules-load.d/ntsync.conf` autoload is
  dropped; ntsync is assert-only (preflight + verify), `PROTON_NO_NTSYNC=1` per-title
  opt-out, mainline ≥ 6.14. The prior "ntsync is now MANAGED (3 autoload confs)" framing
  is retired.
- **MangoHud `cpu_temp` commented out** (v7.76.1) — ships as `# cpu_temp` pending
  per-host hwmon resolution; re-enable is `cpu_custom_temp_sensor=k10temp`. 17 active
  directives.
- **New linux-firmware preflight advisory** (v7.76.0) — hard-warn on a `20251125*` MES
  blob (gfx1151 GCVM_L2 hang), soft-warn below `20260110`.
- **New RTC write-back** (`_ry_rtc_writeback`, v7.74.1) — `hwclock --systohc --utc`
  after NTP sync so a skewed RTC stops poisoning timer persistence stamps.
- **Governor/EPP pairing is LIVE** (`powersave` + `balance_performance`, v7.75.1) —
  shipped value asserted by both cpupower and the udev rule; still under the governor/EPP
  special case.
- **Verify subsystem re-architected** — 12 orchestrators with new `_vrk_*`/`_vrkm_*`/
  `_vrsv_*`/`_kb_*` families alongside `_vss_*`/`_vre_*`/`_vrs_*`.

## What's new in this revision (deeper pass)

- **§13 Candidate enhancements (gaming-first)** — a new investigation section auditing
  knobs the profile does **not** set, each as an ADD-default / ADD-opt-in /
  KEEP-omitted decision: `mitigations=off`, `amdgpu.ppfeaturemask`, `preempt=full`
  (13a); `RADV_PERFTEST`/`RADV_DEBUG`/present-mode (13b); DXVK GPL + compiler-threads,
  upscaler envs, `VKD3D_CONFIG` (13c); Resizable BAR / SAM verification, gaming-vs-compute
  GTT verdict (13d); `read_ahead_kb`/`nr_requests`, `vm.max_map_count` exactness,
  `isolcpus` KEEP-omitted (13e). Bias is KEEP-omitted unless the gaming win is concrete
  and low-risk.
- **Value-anchored items** previously generic: zstd `COMPRESSION_OPTIONS=(-1 -T0)`
  (negative level — ESP-budget vs boot-time), an EPP=performance gaming opt-in, the
  un-deployed scheduler ATTRs, and the GameMode-integration gap (no `gamemoderun` /
  `gamemode.ini`).

## Structure

The prompt is organized as:

- **What changed since v7.75.1** — the re-audit focus list (re-validate these; do not
  carry forward prior conclusions).
- **Mission / Rules / Output** — review mandate, flag-don't-fix discipline, required
  report shape (now including a separate §13 candidate-enhancement matrix).
- **VERIFY block** — post-reboot `fish` assertions; hard asserts (incl. the inverted
  `amd_iommu=off`/0-groups and `_vrkm_iommu`) vs warn-level checks.
- **Security delta** — ordered list of where the profile reduces hardening vs CachyOS
  defaults (now with `amd_iommu=off` at position 2).
- **Investigation §1–§12** — ordered by installer phase: platform baseline, packages,
  kernel cmdline, bootloader/initramfs, GPU/Vulkan, CPU power, memory/storage,
  network/latency, systemd units + time-sync, security, known issues/DKMS,
  MangoHud/Bluetooth/hygiene.
- **Investigation §13** — candidate enhancements (absent knobs, gaming-first).
- **Scope & non-goals** — out-of-scope surfaces and the reinstatement rule.
- **Deep-pass appendix (§A–§F)** — artifact-level layer beneath §1–§13: the install-phase
  model (§A), the exact rendered file bodies to validate rule-by-rule (§B), the complete
  `--verify` subsystem across 12 orchestrators (§C), the fstab normalization logic (§D),
  preflight gate ordering + exit codes (§E), and the deeper-pass actionable deltas (§F).
- **Deepest-pass appendix (§G–§L)** — robustness & correctness audit surface (does the
  installer run *safely*, independent of what it configures): atomic-write guarantees
  (§G), instance-lock & PID-recycle TOCTOU (§H), privilege handling (§I), boot-wipe gate
  & boot-critical rollback (§J), signal & exit teardown (§K), pacman transaction safety
  (§L). Closes with a required **ROBUSTNESS verdict**; any GAP in
  atomic-write/lock/boot-wipe is release-blocking and outranks every tuning finding.

## Output contract

A reviewer answering this prompt must return:

1. **Findings matrix** — box-drawn Unicode, code-fenced, grouped by section:
   ITEM · CURRENT · CALL (KEEP/TUNE/FIX/UNCERTAIN) · RECOMMENDED · IMPACT · RISK ·
   EVIDENCE (URL + version/date/commit).
2. **Candidate-enhancement matrix (§13)** — ITEM · PRESENT?(no) ·
   CALL (ADD-default/ADD-opt-in/KEEP-omitted) · IMPACT · RISK · EVIDENCE.
3. **Before→after** for each TUNE/FIX/ADD — exact current string, exact replacement,
   in-script global.
4. **VERIFY block** — the post-reboot command set.
5. **Security delta vs CachyOS defaults** — ordered.
6. **Verdict** — one per section (OPTIMAL/TUNE/FIX) plus overall
   (PASS/PASS-WITH-FIXES/FAIL).
7. **ROBUSTNESS verdict** — §G–§L (PASS/GAP/UNCERTAIN), separate from tuning verdicts.
8. **Methodology** — source list with access dates/versions; conflicts flagged;
   unknowns marked UNCERTAIN.

## Review rules (summary)

- Hardware-anchored to gfx1151 / Zen 5 / RDNA 3.5 / CachyOS / 128 GB unified /
  dual 10 GbE.
- Deliberate trade-offs: **flag + quantify, do not auto-FIX.** FIX is reserved for
  incorrect/superseded/deprecated/harmful values. §13 uses ADD/KEEP-omitted instead.
- Rate IMPACT × RISK. Default KEEP (or KEEP-omitted in §13) when impact is marginal and
  risk is non-trivial.
- Never invent params/flags/keys/options/URLs — cite a source or mark UNCERTAIN.
- Flag every source conflict and state which is trusted.
- Exact versions (kernel / Mesa / linux-firmware / pkg) and exact before→after,
  mapped to the in-script global.

## Special-case rules

Config choices protected from being flagged as regressions unless current upstream
directly contradicts the rationale:

- **IOMMU (INVERTED)** — the profile now ships `amd_iommu=off` (AMD-Vi fully disabled).
  Do **not** recommend re-adding `iommu=pt`/`amd_iommu=on` as a default unless ROCm/compute
  on gfx1151 is proven to require the IOMMU, or a DMA-isolation requirement is
  established for this single-user desktop. The VFIO/SR-IOV opt-in
  (`amd_iommu=on iommu=pt`) is the documented per-user override, not the profile default.
- **Governor/EPP** — `powersave` + `balance_performance` is the EPP-honoring config
  under `amd_pstate=active`. Do not flag `powersave` as a regression without proving the
  `performance` governor would override the EPP hint. (An EPP=performance *opt-in* is
  evaluated in §6/§13, not a default FIX.)
- **GPU_DPM_LEVEL** — `auto` is deliberate (stop pinning SCLK and stealing Zen 5 boost).
  Do not flag `auto` as a GPU-perf regression without proving `high` materially improves
  frametime/1%-lows without costing CPU boost on the shared 140 W package.

Re-added/live config, no longer "deliberately removed" — evaluate as live config (KEEP
if upstream-valid, FIX-to-remove if inert/harmful): PROTON_FSR4_RDNA3_UPGRADE, MangoHud
gpu_power/text_outline/toggle_hud.

Still deliberately removed (reinstatement rule applies): `amdgpu.ppfeaturemask`
(re-evaluated in §13a as an undervolt/OC opt-in, not a silent default), `--country`
flag, TTM/GTT cap, RADV drirc, MangoHud cpu_temp/fps_metrics,
vm.page-cluster/vm.vfs_cache_pressure (vendor-provided), ntsync modules-load.d autoload
(now assert-only).

## Non-goals

Recommendations only — no modified script emitted. Out of scope: dotfiles, shells,
editors, secrets, backups, multi-user, non-CachyOS, laptops, UKI. Per-game Proton tuning
is secondary to system-wide config; the prompt leans gaming over AI/compute.

## Usage

Provide this prompt together with the matching `ry-install.fish` v7.77.1 to a
research-capable model with live source access. When the script version changes, fork
the prompt to a new version file and update the counts, the inline v-deltas, and the
VERIFY/security-delta sections to match.

## See also

`CHANGELOG.md` — prompt revision history.
