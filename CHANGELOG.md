# Changelog

All notable changes to the ry-install deep-research prompt.

Format follows the kernel.org convention: newest first, grouped by change class.
The prompt version tracks the `ry-install.fish` version it audits.

## [v7.70.1] - 2026-06-26

Target script: `ry-install.fish` v7.70.1.

### Changed

- Editorial pass on the prompt: removed conversational filler and telegraphic
  asides, normalized section headers and list phrasing to professional register,
  and expanded cramped parentheticals into readable clauses. No technical content
  changed — all 11 counts, every config value-string, the VERIFY block, the
  security-delta ordering, all `§1`–`§12` investigation items, every source list,
  and both r8169 commit hashes are byte-for-byte preserved in meaning. Verified by
  token-presence check against the prior revision.
- Pinned all version-floor language and inline deltas to v7.70.1 as the script
  under audit; source-of-truth order set to script > README > CHANGELOG.

### Profile facts carried (vs v7.70.0)

- VERIFY block hardened to assert `scaling_governor=powersave` and
  `EPP=balance_performance` as the EPP-honoring config under `amd_pstate=active`,
  rather than expecting the `performance` governor.
- Investigation §6 marked the governor/EPP pairing as a protected special case:
  `powersave` must not be flagged as a regression without proving the `performance`
  governor would override the EPP hint.
- Security-delta ordering retained: UMIP off (`clearcpuid=514`) listed first as the
  headline open reduction, followed by `split_lock_detect=off`, plaintext DNS,
  firewall tightening (net positive), and IOMMU restored (closed reduction).

## [v7.70.0] - prior

Target script: `ry-install.fish` v7.70.0. Baseline for the deltas referenced
throughout the v7.70.1 prompt.

### Added

- Bluetooth `main.conf` introduced: `FastConnectable=true`, `AutoEnable=true`,
  `ReconnectAttempts=3`.
- `rtkit` and `modemmanager` mask added to the package/units posture.

### Changed

- Soft Mesa floor raised 25.3 → 26.0 (preflight warn).
- Bluetooth `ReconnectAttempts` lowered 7 → 3.

### Removed

- Second regdomain file `conf.d/wireless-regdom` removed; single regdom file
  `/etc/iw-regdomain` (`COUNTRY=US`) is now authoritative. VERIFY no longer checks
  `conf.d/wireless-regdom`.
- Bluetooth explicit `ReconnectIntervals` list (1,2,4,8,16,32,64) dropped in favor of
  BlueZ default backoff.
- MangoHud directives `gpu_power`, `cpu_temp`, `text_outline`, `toggle_hud`,
  `fps_metrics` removed (15 directives remain).
- RADV drirc (`95-ry-radv-apu.conf`) removed — gfx1151 reports `uma:1` natively.
- TTM/GTT module params removed — kernel ≥6.16.9 auto-sizes GTT to ~50%-of-RAM
  (~62 GiB).
- Boot-time / THP / KSM checks removed from VERIFY; `ttm.*` / drirc /
  `radv_enable_unified_heap_on_apu` globals no longer exist and are not verified.
- Deliberately removed values now under the reinstatement rule:
  amdgpu.ppfeaturemask, PROTON_FSR4_RDNA3_UPGRADE, `--country` flag, TTM/GTT cap.

### Security

- IOMMU: `amd_iommu=off` removed; current `iommu=pt` (AMD-Vi on) — prior reduction
  recorded as closed.
- Wi-Fi backend reverted to `wpa_supplicant` (from iwd) as an MT7925-stability move;
  removed iwd `main.conf` leaves no dangling reference.

---

### Notes

- This changelog documents revisions to the **prompt**, not to `ry-install.fish`.
  The script maintains its own CHANGELOG, which is the authoritative source for
  profile changes; entries above summarize only the deltas the prompt explicitly
  references.
- When the script version advances, fork the prompt to a new version file and add a
  corresponding entry here. Historical entries are never renumbered.
