# Changelog

All notable changes to the ry-install deep-research prompt.

Format follows the kernel.org convention: newest first, grouped by change class.
The prompt version tracks the `ry-install.fish` version it audits.

## [v7.77.1] - 2026-06-28

Target script: `ry-install.fish` v7.77.1. Forked from the v7.75.1 prompt; counts,
inline v-deltas, VERIFY block, security-delta ordering, and the verify-subsystem map
re-pinned to the current script. A leading "What changed since v7.75.1" section
enumerates every delta the reviewer must re-evaluate. This revision also adds a
deeper-pass gaming-first investigation layer (§13) and value-anchors several previously
generic items.

### Validated (Round 2 live-source pass — 2026-06-28; prompt-only, script unchanged at v7.77.1)

Two rounds of current-upstream validation folded into the prompt. Open questions now
carry the verified answer inline (marked **RESOLVED (Round 2)**); no script change, so
the prompt version stays v7.77.1 (this changelog tracks the script version it audits).

#### Corrected (factual anchors that were wrong)

- **Kernel floor rationale** — `KERNEL_MIN=6.18` re-anchored from "RTL8127 networking" to
  **gfx1151 GPU/ROCm stability** (avoid < 6.18.4; 6.18.6+ recommended). The two r8169
  commits are PRIMARY-VERIFIED and land earlier: `f24f7b2f3af9` ("r8169: add support for
  RTL8127A") in mainline **6.16**; `ae1737e7339b` ("r8169: fix RTL8127 hang on
  suspend/shutdown", `Cc: stable`, `Fixes: f24f7b2f3af9`) in mainline **6.18**,
  backported **6.17-stable**. NIC floor is therefore 6.17; 6.18 is justified by the iGPU.
  Updated in the header hard-floors line, §1, §11.
- **linux-firmware advisory semantics** — `20260110` is a **revert**, not a new fix.
  `20251125` shipped the regressed GC 11.5.1 microcode (gfx1151 MES → 0x83, commit
  `9643cbf2…`); `20260110` reverted to stable MES 0x80 (`c092c7487e…`). Known-good:
  20251111 (0x80) or 20260110+. The stale "MES 0x86" label is removed (relevant versions
  are 0x83 bad / 0x80 good). Latest release 20260624. Updated in header, §1, §11,
  appendix §E/§F, VERIFY block.
- **MangoHud sensor key** — corrected `cpu_custom_temp_sensor` → **`cpu_temp_sensor`**
  (MangoHud #1825) across all five sites (What-changed item 4, §12, appendix §B, §F,
  VERIFY block). Added the open caveat that enabling `cpu_temp` zeroes `cpu_power` on
  Zen 5 (#1794). `k10temp`/Tctl confirmed the correct Zen 5 source.

#### Resolved (open questions answered, no text was wrong — just under-determined)

- **amd_iommu=off does NOT break ROCm** on gfx1151 (`rocminfo` reports `IOMMU Support:
  None`; the Strix Halo LLM-toolbox community recommends `amd_iommu=off` for the unified
  pool). Removed the compute-escalation gate in §3/§5/§10/§F — accepted security-vs-
  latency trade-off, not a compute regression.
- **§13 candidate knobs fully resolved** with CALL + evidence:
  - KEEP-omitted: `mitigations=off` (Zen 5 not Inception-affected; no gaming benefit
    measured), `amdgpu.ppfeaturemask` (GPU OC/undervolt unimplemented on gfx1151; real
    lever is CPU ryzenadj), **`preempt=full` (redundant — CachyOS desktop default is
    already `full`)**, `nvme_core.io_timeout`/`pcie_port_pm`, `RADV_PERFTEST` gpl/sam
    (gpl default since Mesa 23.1, sam auto-on for APU), `RADV_DEBUG`/present-mode/
    `mesa_glthread`, DXVK GPL + numCompilerThreads (auto-optimal; async superseded),
    `VKD3D_CONFIG`, `read_ahead_kb`/`nr_requests` (defaults optimal), `vm.max_map_count`
    (2147483642 sufficient), `isolcpus`/`nohz_full`/`rcu_nocbs`, **GameMode (redundant —
    governor/EPP/DPM already pinned profile-wide)**.
  - UNCERTAIN (no gfx1151/Zen-5 data exists — do not ADD on a guess): RADV `nggc`;
    EPP `balance_performance`→`performance` gaming delta (§6).
  - verify-only/auto: ReBAR/SAM auto-on for the APU (optional ReBAR-off INFO).
- **amd_pstate EPP triple CONFIRMED** — active + powersave + balance_performance is the
  documented EPP-honoring max-perf config on Zen 5; governor not flipped.
- **MT7925 status** sharpened to "improving, not closed" (mt7925e fixes in 6.17+, already
  under the 6.18 floor; drop-in stays defensive).
- **ACP70 internal-mic gap CONFIRMED still open** upstream (no machine driver / UCM
  profile as of mid-2026) — documented as a known hardware gap, not a recommendable floor.

#### Still open after Round 2 (marked UNCERTAIN in the prompt)

- Exact linux-firmware release re-landing the corrected GC 11.5.1 MES blob
  (`a0f0e52138…`) — soft-warn kept pinned to 20251125.
- sdboot-manage upstream maintenance cadence.
- Verbatim author/AuthorDate for the three firmware commit hashes (git.kernel.org /
  gitlab blocked automated fetch; hashes secondary-corroborated via Launchpad #2129150,
  Debian firmware-nonfree changelog, ROCm #5724).



- **§13 Candidate enhancements** — a new investigation section auditing knobs the
  profile does NOT set, each as an ADD-default / ADD-opt-in / KEEP-omitted decision with
  IMPACT × RISK. Subsections:
  - §13a kernel cmdline: `mitigations=off` (Zen 5 cost vs the already-accepted UMIP-off
    threat model), `amdgpu.ppfeaturemask` (undervolt/OC via CoreCtrl/LACT),
    `preempt=full` (frame-pacing, gated on the CachyOS default PREEMPT_DYNAMIC mode),
    `nvme_core.io_timeout`/`pcie_port_pm`.
  - §13b RADV/Mesa: `RADV_PERFTEST` (gpl/sam), `RADV_DEBUG`, present-mode/`vblank_mode`,
    `mesa_glthread` — each as default-vs-win.
  - §13c DXVK/VKD3D-Proton: GPL + compiler-threads (with the explicit "async→GPL,
    do-not-recommend-old-async" note), upscaler envs, `VKD3D_CONFIG` dxr.
  - §13d firmware/platform: Resizable BAR / Smart Access Memory verification (proposed
    advisory INFO for ReBAR-off), gaming-vs-compute GTT verdict.
  - §13e scheduler/memory: `read_ahead_kb`/`nr_requests` (un-deployed), `vm.max_map_count`
    exact-requirement confirm, `isolcpus`/`nohz_full`/`rcu_nocbs` KEEP-omitted with
    rationale.
- Bias documented as KEEP-omitted unless the gaming win is concrete and low-risk.
- All §13 "absent" claims fact-checked against `ry-install.fish` v7.77.1 (probe
  confirmed `mitigations`/`RADV_PERFTEST`/`DXVK_ASYNC`/`gamemode`/`rebar`/`read_ahead_kb`
  all absent from source).

### Added (value-anchored sharpening of existing items)

- §4: zstd surfaced as the literal `COMPRESSION_OPTIONS=(-1 -T0)` (negative level =
  fastest/lowest-ratio, not default-3) — ESP-budget vs boot-time trade with the
  `BOOT_SPACE_*` gates, replacing the generic "confirm zstd level."
- §6: EPP item now demands a quantified `balance_performance` → `performance` gaming
  opt-in (udev EPP ATTR only; governor stays powersave).
- §7: `nr_requests`/`read_ahead_kb` documented as NOT deployed (only `scheduler=none`
  is) with an ADD-or-KEEP decision, replacing "evaluate nr_requests and read_ahead_kb."
- §3: preempt item tied to the actual `_vrk_cmdline` dmesg `Dynamic Preempt:` detection.
- §2: added the GameMode-integration gap (no explicit `gamemode` pkg, no `gamemode.ini`,
  no `gamemoderun` env; static profile may already cover GameMode's governor switch).
- §5: ntsync 6.14-mainline vs 6.18-floor consistency confirm.

### Added (re-architected verify subsystem — §C rewrite)

- §C rewritten to the real v7.77.1 architecture: 12 orchestrators across six sub-families
  — `_vss_*` (static on-disk), `_kb_*` (5 known-benign INFO), `_vrk_*` (runtime-kernel:
  cmdline/gpu/cpu/module/clocksource), `_vrkm_*` (module-state: amdgpu/**iommu**/blacklist),
  `_vrsv_*` (runtime-services), `_vre_*` (runtime-env), `_vrs_*` (runtime-session). The
  prior `_vss(11)/_vre(7)/_vrs(5)`-only map is retired.
- `_vrk_cmdline` documented as a generic loop asserting EVERY `KERNEL_PARAMS` token +
  `rw` (so `amd_iommu=off` is auto-asserted).
- `_vrkm_iommu`, `_vrk_clocksource` (TSC-demotion correlation), and `_vrkm_blacklist`
  added to the verify surface.

### Changed (IOMMU inversion — the headline delta)

- IOMMU premise INVERTED a second time: `iommu=pt` → `amd_iommu=off` (v7.77.0).
  §3 re-audits AMD-Vi-off on merits (incl. a ROCm/SVM-breakage escalation gate),
  §10 + security-delta reorder it to the #2 OPEN reduction, the IOMMU special case flips
  (do not re-add `iommu=pt` as a default), and the VERIFY block + §C swap the
  `iommu=pt` assert for `amd_iommu=off`/0-groups via `_vrkm_iommu`.
- Counts re-pinned to all 19 hard-asserted globals; KERNEL_PARAMS stays 16 (IOMMU swap),
  NTSYNC_MODLOAD_CONFS removed from the manifest, MangoHud 17 active (+1 commented).
- §5: ntsync framing changed from MANAGED (3 autoload confs) to assert-only;
  `PROTON_NO_NTSYNC=1` opt-out documented; the "largest unmanaged surface" framing
  retired.
- §12 / §B9: MangoHud `cpu_temp` documented as commented out (re-enable via
  `cpu_temp_sensor=k10temp`; key corrected from `cpu_custom_temp_sensor` in the Round 2
  pass).
- §1 / §11: linux-firmware preflight advisory added (`20251125*` hard-warn,
  `< 20260110` soft-warn).
- §9 / §A: RTC write-back (`_ry_rtc_writeback`) added to the Services phase.
- VERIFY block: asserts for `amd_iommu=off`/0-groups, `pacman -Q linux-firmware`,
  `_vrk_cmdline` generic param loop; GPU DPM assert stays `auto`.
- Security-delta head: UMIP off first, then `amd_iommu=off` (INVERTED, now open), then
  `split_lock_detect=off`, plaintext DNS, remote-play opt-in, firewall default-deny.
- Scope/non-goals: reinstatement rule cross-linked `amdgpu.ppfeaturemask` to §13a;
  ntsync autoload moved into the deliberately-removed set.

### Carried unchanged (vs v7.75.1)

- Governor/EPP special case retained (`powersave` + `balance_performance` is the
  EPP-honoring config under `amd_pstate=active`).
- GPU_DPM_LEVEL special case retained (`auto`, un-pinned SCLK).
- nftables ruleset, udev rules, remote-play ports, fstab normalization (§D), preflight
  gates (§E), and the entire §G–§L robustness layer verified byte/behavior-identical;
  all robustness machinery (atomic-write, lock, boot-wipe, signal teardown, pacman) and
  the ROBUSTNESS verdict carried forward, re-scoped to §1–§13.
- Naming FIX (Low/Low) retained: human-facing "GTR9 Pro" vs internal `gtr_pro`.

## [v7.75.1] - 2026-06-27

Target script: `ry-install.fish` v7.75.1. Forked from the v7.70.1 prompt; counts,
inline v-deltas, VERIFY block, and security-delta ordering re-pinned to the current
script. A leading "What changed since v7.70.1" section enumerates every delta the
reviewer must re-evaluate.

### Added (deepest pass — robustness & correctness audit §G–§L)

- §G atomic-write guarantees, §H instance-lock & PID-recycle TOCTOU, §I privilege
  handling, §J boot-wipe gate & boot-critical rollback, §K signal & exit teardown,
  §L pacman transaction safety, plus the required ROBUSTNESS verdict block. All §G–§L
  mechanism names, exit codes, and helper signatures fact-checked present (32/32).

### Added (deep pass — artifact-level appendix §A–§F)

- §A install-phase model, §B exact rendered file bodies, §C full `--verify` subsystem,
  §D fstab normalization, §E preflight gate ordering, §F 10 actionable deltas. All §A–§F
  quoted strings fact-checked present (21/21).

### Added (first pass)

- "What changed since v7.70.1" section; §3 four new kernel params; §5 ntsync-as-MANAGED
  + PROTON_FSR4_RDNA3_UPGRADE re-eval; §8/§11 mt7925e drop-in; §10 RY_REMOTE_PLAY_PORTS;
  §12 `_vss_known_benign` advisory sub.

### Changed

- Counts re-pinned: KERNEL_PARAMS 12 → 16, ENV_VARS 10 → 11, SYSCTL 11 → 9, managed
  files 17 → 18, MangoHud 15 → 18, NTSYNC_MODLOAD_CONFS 3 added; KERNEL_MIN 6.18 hard
  floor; GPU clock-floor `high` → `auto`; MangoHud directives restored;
  vm.page-cluster/vm.vfs_cache_pressure dropped as vendor duplicates.

## [v7.70.1] - 2026-06-26

Target script: `ry-install.fish` v7.70.1.

### Changed

- Editorial pass: removed conversational filler, normalized headers and list phrasing to
  professional register. No technical content changed. Pinned version-floor language and
  inline deltas to v7.70.1; source-of-truth order script > README > CHANGELOG.

### Profile facts carried (vs v7.70.0)

- VERIFY hardened to assert `scaling_governor=powersave` and `EPP=balance_performance`;
  §6 governor/EPP special case; security-delta ordering retained.

## [v7.70.0] - prior

Target script: `ry-install.fish` v7.70.0. Baseline for the v7.70.1 deltas.

### Added

- Bluetooth `main.conf` (FastConnectable/AutoEnable/ReconnectAttempts); `rtkit` +
  `modemmanager` mask.

### Changed

- Soft Mesa floor 25.3 → 26.0; Bluetooth ReconnectAttempts 7 → 3.

### Removed

- Second regdomain file; Bluetooth explicit ReconnectIntervals; MangoHud
  gpu_power/cpu_temp/text_outline/toggle_hud/fps_metrics (some later restored); RADV
  drirc; TTM/GTT module params; boot-time/THP/KSM checks from VERIFY.

### Security

- IOMMU: `amd_iommu=off` removed; `iommu=pt` (AMD-Vi on). NOTE: subsequently
  re-inverted to `amd_iommu=off` in v7.77.0 — see the v7.77.1 entry. Wi-Fi backend
  reverted to `wpa_supplicant`.

---

### Notes

- This changelog documents revisions to the **prompt**, not to `ry-install.fish`.
  The script maintains its own CHANGELOG, which is the authoritative source for profile
  changes; entries above summarize only the deltas the prompt explicitly references.
- When the script version advances, fork the prompt to a new version file and add a
  corresponding entry here. Historical entries are never renumbered.
