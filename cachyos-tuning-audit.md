#  cachyos-tuning-audit — ry-install Tuning (CachyOS · Beelink GTR9 Pro)

Target: `ry-install.fish` **v7.77.1** (attached). Source of truth: script > README > CHANGELOG.

**Platform:** Beelink GTR9 Pro · Ryzen AI Max+ 395 "Strix Halo" (Zen 5, 16C/32T, gfx1151) · Radeon 8060S (40 RDNA 3.5 CUs) · XDNA 2 NPU · 128 GB LPDDR5X-8000 unified (≤96 GB as VRAM) · dual M.2 NVMe (ext4) · dual 10 GbE (RTL8127) + Wi-Fi 7 (MT7925) + BT 5.4 · 140 W TDP · CachyOS · systemd-boot.

**Counts (from `_ir_validate_counts`, all 19 hard-asserted):** KERNEL_PARAMS 16 · MKINITCPIO_HOOKS 11 · MKINITCPIO_MODULES 1 · LOGIND_IGNORE_KEYS 8 · ENV_VARS 11 · SYSCTL_VALUES 9 · PKGS_ADD 17 · PKGS_DEL 9 · MASK 10 · EXPECTED_VULKAN_PKGS 2 · EXPECTED_SERVICES 5 · _RY_PKG_MANAGED_SERVICES 1 · _RY_POST_HOOKS 18 · _RY_BOOT_CRITICAL_DSTS 4 · _RY_PHASE_NAMES 6 · _RY_BACKUP_TARGETS 2 · _RY_TMPDIR_GLOBS 6 · SYSTEM_DESTINATIONS 15 · USER_DESTINATIONS 3. Managed files = 18 (15 system + 3 user). MangoHud.conf = 17 active directives + 1 commented (`cpu_temp`). NTSYNC autoload confs = **0** (de-managed v7.76.1; ntsync is assert-only).

**Hard floors:** KERNEL_MIN **6.18** (preflight hard-fail) — anchored to **gfx1151 GPU/ROCm stability**, NOT networking: kernels < **6.18.4** carry a gfx1151 stability bug, **6.18.6+** recommended. The RTL8127 anchors are satisfied earlier (PRIMARY-VERIFIED: r8169 RTL8127A support `f24f7b2f3af9` landed mainline **6.16**; suspend/shutdown-hang fix `ae1737e7339b`, `Cc: stable`, landed mainline **6.18** and was backported to **6.17-stable**), so the NIC floor is 6.17 and 6.18 is conservative for networking but justified for the iGPU. · CPU gate `Ryzen AI Max` · soft Mesa < 26.0 warn · linux-firmware advisory (hard-warn on `20251125*` MES blob — regressed GC 11.5.1 MES 0x83, commit `9643cbf2…`; soft-warn < `20260110` — the **revert** release `c092c7487e…` back to stable MES 0x80; non-fatal).

## What changed since v7.75.1 (re-audit focus)

The prior prompt targeted v7.75.1. The following are NEW or INVERTED and need fresh evidence — do not carry forward the old conclusions on these:

1. **IOMMU INVERTED a second time: `iommu=pt` → `amd_iommu=off`** (v7.77.0). AMD-Vi is now **fully disabled**, not "restored." KERNEL_PARAMS stays 16 (one-for-one swap). The profile is declared a single-purpose gaming/LLM desktop with no PCI passthrough. Every prior "IOMMU restored / AMD-Vi on / closed reduction" statement is now wrong. Re-evaluate `amd_iommu=off` on its own merits (latency/throughput vs the security and ROCm/USB4-isolation cost), and confirm the README's VFIO/SR-IOV escape hatch (`amd_iommu=on iommu=pt`) is the correct opt-back-in.
2. **NEW verify sub `_vrkm_iommu`** (v7.77.0) — derives expected IOMMU state from KERNEL_PARAMS (`amd_iommu=`/`intel_iommu=`/`iommu=`) and asserts it against live `/sys/kernel/iommu_groups` count + `dmesg` AMD-Vi/DMAR correlation. With `amd_iommu=off` it **hard-fails** if any iommu_groups are present (BIOS/firmware override). Confirm the 0-groups expectation is right and the dmesg grep patterns are current.
3. **ntsync DE-MANAGED: autoload conf dropped** (v7.76.1). `/etc/modules-load.d/ntsync.conf` is gone; ntsync is now **assert-only** (preflight + `_vss_ntsync_modules` + `_vre_ntsync`). The `builtin|loaded|loaded_nodev|missing` state machine (`_ntsync_state`) survives but no longer *deploys* anything. README documents mainline ≥ 6.14 and a per-title opt-out `PROTON_NO_NTSYNC=1`. The prior "ntsync is now MANAGED (3 autoload confs)" framing is fully retired — confirm assert-only is correct given the CachyOS kernel ships `CONFIG_NTSYNC=y`.
4. **MangoHud `cpu_temp` COMMENTED OUT** (v7.76.1, after being added in v7.76.0). The line ships as literal `# cpu_temp` pending per-host hwmon resolution; the documented re-enable is **`cpu_temp_sensor=k10temp`** (`k10temp`/Tctl is the correct Zen 5 hwmon source). **RESOLVED (Round 2):** the config key is `cpu_temp_sensor`, NOT `cpu_custom_temp_sensor` (MangoHud #1825). CAVEAT — enabling `cpu_temp` currently zeroes `cpu_power` on Zen 5 (MangoHud #1794, still open), so re-enabling both together loses the CPU-power readout until that bug lands. 17 active directives.
5. **NEW linux-firmware preflight advisory** (v7.76.0) — hard-warn (non-fatal) on a `20251125*` MES blob (gfx1151 GCVM_L2 GPU hang), soft-warn below `20260110`. **RESOLVED (Round 2, commit-pinned):** `20251125` shipped the regressed GC 11.5.1 microcode that pushed gfx1151 MES to version **0x83** (commit `9643cbf2…`, "amdgpu: update gc 11.5.1 firmware") — confirmed to break ROCm on gfx1151. `20260110` did NOT introduce a new fix; it is the **revert** release (`c092c7487e…`, "Revert 'amdgpu: update GC 11.5.1 firmware'") back to stable MES **0x80**. Known-good fallbacks: **20251111** (MES 0x80) or **20260110+**. A corrected GC 11.5.1 re-introduction for gfx1150/1151 exists (`a0f0e52138…`) but its exact release tag is unconfirmed; keep the soft-warn anchored to 20251125. Latest release as of audit: **20260624**. (Firmware commit hashes are secondary-corroborated via Launchpad #2129150, the Debian firmware-nonfree changelog, and ROCm #5724 — git.kernel.org/kernel-firmware blocked direct fetch.)
6. **NEW RTC write-back** (`_ry_rtc_writeback`, v7.74.1) — after NTP sync confirms, runs `hwclock --systohc --utc` (skipped if `RTCInLocalTZ=yes` or hwclock absent; non-fatal) so a skewed RTC stops poisoning systemd-timer persistence stamps. Confirm the UTC-only write-back is safe and that deferring local-RTC to systemd is correct.
7. **Governor/EPP pairing is now LIVE** (`powersave` + `balance_performance`, landed v7.75.1). Treat under the governor/EPP special case — do not flag `powersave` as a regression — but it is no longer a "carried" claim; it is the shipped value asserted by both cpupower and the udev rule. Confirm the active + powersave + balance_performance triple is the documented EPP-honoring max-perf config on Zen 5.
8. **Verify subsystem re-architected.** The `--verify` path is now 12 orchestrators with new sub-families: `_vrk_*` (runtime-kernel: cmdline/gpu_state/cpu_state/module_state/clocksource), `_vrkm_*` (module-state subs: amdgpu/iommu/blacklist), `_vrsv_*` (runtime-services), `_kb_*` (5 known-benign INFO subs), plus the original `_vss_*`/`_vre_*`/`_vrs_*`. Section C below is rewritten to the real architecture — the prior `_vss_*(11)/_vre_*(7)/_vrs_*(5)`-only map was incomplete.

Two doc-only banners were added in v7.77.1 (`VERIFY-STATIC: KNOWN-BENIGN ADVISORIES`, `VERIFY-RUNTIME: MODULE-STATE SUBS`) — comment-only, no behavior change; ignore for tuning.

This revision also adds **§13 (Candidate enhancements)** — a gaming-first audit of knobs the profile does NOT set (`mitigations=off`, `RADV_PERFTEST`, ReBAR/SAM, `preempt=full`, DXVK GPL, `read_ahead_kb`, et al.), each as an ADD-default / ADD-opt-in / KEEP-omitted decision — and value-anchors several previously generic items (zstd `COMPRESSION_OPTIONS=-1 -T0`, EPP=performance opt-in, un-deployed scheduler ATTRs, GameMode integration gap).

> **Round 2 corrections applied (2026-06-28).** Two rounds of live-source validation have been folded into this prompt; items marked **RESOLVED (Round 2)** carry the verified answer inline rather than an open question. Headline corrections: (1) the **6.18 floor** is re-anchored to gfx1151 stability (avoid < 6.18.4; 6.18.6+ recommended) — the RTL8127 r8169 commits land at 6.16 (`f24f7b2f3af9`) and 6.18/6.17-stable (`ae1737e7339b`), so the NIC floor is 6.17, not 6.18; (2) the **linux-firmware** soft-warn is a **revert**, not a new fix — `20251125` shipped bad GC 11.5.1 MES 0x83 (`9643cbf2…`), `20260110` reverted to MES 0x80 (`c092c7487e…`), known-good 20251111 / 20260110+; (3) **MangoHud** key is `cpu_temp_sensor` (not `cpu_custom_temp_sensor`), with an open Zen 5 bug zeroing `cpu_power`; (4) **amd_iommu=off does NOT break ROCm** on gfx1151 (`IOMMU Support: None`) — accepted trade-off, no compute escalation; (5) all **§13 candidate knobs resolved** — 6 KEEP-omitted (incl. `preempt=full` redundant, GameMode redundant), 2 UNCERTAIN (RADV `nggc`, EPP=performance gaming). **Still open after Round 2 (mark UNCERTAIN):** the exact linux-firmware release re-landing the corrected GC 11.5.1 blob (`a0f0e52138…`); sdboot-manage maintenance cadence; ACP70 internal-mic ASoC machine driver (confirmed still absent upstream — a known hardware gap the profile cannot fix). Firmware commit hashes are secondary-corroborated (Launchpad #2129150 / Debian changelog / ROCm #5724); git.kernel.org/gitlab blocked direct fetch.

## Mission

Evaluate every config decision against current upstream sources for this exact silicon. Return a prioritized, evidence-backed tuning report. The profile deliberately trades PCI-passthrough capability, power-saving, and host firewalling for performance and latency; confirm each choice is current and correct, surface anything superseded or harmful, and quantify safety deltas without second-guessing intentional design.

## Rules

1. Item-by-item, hardware-anchored to gfx1151 / Zen 5 / RDNA 3.5 / CachyOS / 128 GB unified / dual 10 GbE.
2. Respect deliberate trade-offs: **flag and quantify, do not auto-FIX.** Reserve FIX for incorrect, superseded, deprecated, or harmful values.
3. Rate IMPACT × RISK (High/Med/Low). Default KEEP when impact is marginal and risk is non-trivial.
4. Never invent params, flags, keys, options, or URLs. Cite a source or mark UNCERTAIN.
5. Flag every source conflict and state which is trusted.
6. Give exact versions (kernel / Mesa / linux-firmware / pkg) and exact before→after, mapped to the in-script global.

## Output

- **Findings matrix** (box-drawn Unicode, code fence, grouped by section): ITEM · CURRENT (v7.77.1 value) · CALL (KEEP/TUNE/FIX/UNCERTAIN) · RECOMMENDED · IMPACT · RISK · EVIDENCE (URL + version/date/commit).
- **Candidate-enhancement matrix** (§13, separate): ITEM · PRESENT?(no) · CALL (ADD-default/ADD-opt-in/KEEP-omitted) · IMPACT · RISK · EVIDENCE.
- **Before→after** for each TUNE/FIX/ADD: exact current string, exact replacement, in-script global.
- **VERIFY block** (post-reboot commands, below).
- **Security delta vs CachyOS defaults** (ordered, below).
- **Verdict:** one per section (OPTIMAL/TUNE/FIX) plus overall (PASS/PASS-WITH-FIXES/FAIL).
- **ROBUSTNESS verdict** (§G–§L; separate from tuning verdicts).
- **Methodology:** source list with access dates and versions; conflicts flagged; unknowns marked UNCERTAIN.

### VERIFY block

```fish
cat /proc/cmdline
cat /sys/.../cpu0/cpufreq/scaling_driver                    # amd-pstate-epp
cat /sys/.../cpu0/cpufreq/scaling_governor                  # powersave (EPP-honoring under pstate=active)
cat /sys/.../cpu0/cpufreq/energy_performance_preference     # balance_performance
cat /sys/.../amd_pstate/status                              # active
cat /sys/.../amd_pstate/prefcore                            # enabled
cat /sys/.../cpufreq/boost                                  # 1
cat /sys/.../clocksource0/current_clocksource               # tsc (HPET → fail; check dmesg TSC-demotion)
cat /sys/block/nvme0n1/queue/scheduler                      # [none] (adjust node)
cat /sys/class/drm/card*/device/power_dpm_force_performance_level   # auto (DPM floor un-pinned; GPU_DPM_LEVEL)
find /sys/kernel/iommu_groups -mindepth 1 -maxdepth 1 -type d | wc -l   # 0 (amd_iommu=off → no groups)
cat /proc/cmdline | rg -o 'amd_iommu=\S+'                   # amd_iommu=off (AMD-Vi disabled — INVERTED from iommu=pt)
cat /proc/cmdline | rg -o 'clearcpuid=\S+'                  # clearcpuid=514 (UMIP off)
cat /proc/cmdline | rg -o 'processor.max_cstate=\S+'        # 1 (C-state cap)
cat /proc/cmdline | rg -o 'fsck\S+'                         # fsck.mode=force fsck.repair=yes
ls -l /dev/ntsync                                           # present (assert-only; autoload conf dropped v7.76.1)
sudo dmesg | rg -i 'AMD-Vi|DMAR'                            # expect NO "AMD-Vi: Enabled" (amd_iommu=off)
cat /etc/modprobe.d/60-ry-mt7925e.conf                      # options mt7925e disable_aspm=1
pacman -Q linux-firmware                                    # good: 20251111 or >=20260110 (revert); AVOID 20251125* (bad MES 0x83); latest 20260624
vulkaninfo | rg -i 'driverName|deviceName'                 # RADV / Radeon 8060S; confirm uma heap
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc vm.max_map_count vm.compaction_proactiveness vm.swappiness
findmnt -no OPTIONS /                                       # noatime,lazytime,commit=10
swapon --show; zramctl                                      # zram active (advisory; profile does not manage zram)
iw reg get | rg -i country                                 # US
cat /etc/iw-regdomain                                       # COUNTRY=US
sudo nft list chain inet filter input                      # policy drop + ICMPv6 NDP/PMTUD + IPv4 diag-only (no inbound ping); +remote-play ports IFF RY_REMOTE_PLAY_PORTS=true
stat -c '%a %U:%G' /etc/NetworkManager/system-connections/* # 0600 root:root
systemctl is-enabled bluetooth.service                     # enabled
printenv MANGOHUD                                          # 1
grep -c '^cpu_temp' ~/.config/MangoHud/MangoHud.conf       # 0 (cpu_temp commented out v7.76.1; re-enable via cpu_temp_sensor=k10temp)
```

Hard `--verify` asserts (mismatch → exit 1): every `KERNEL_PARAMS` token present in `/proc/cmdline` + `rw` (generic loop in `_vrk_cmdline`); scaling_driver=amd-pstate-epp, scaling_governor=powersave, EPP=balance_performance, amd_pstate status/prefcore, boost, clocksource=tsc; GPU `power_dpm_force_performance_level=auto`; **IOMMU effect (`_vrkm_iommu`): with `amd_iommu=off`, 0 iommu_groups required — hard-fail if groups present**; usbcore.autosuspend=-1, nvme_core.default_ps_max_latency_us=0, zswap.enabled∈{N,0}, nmi_watchdog=0, NVMe sched `[none]`; regdom (`/etc/iw-regdomain`); nftables `nd-neighbor-solicit` present (`_vss_nft` / `_vrsv_nft_assert_ndp`); mt7925e `disable_aspm=1`; NM system-connections 0600 root:root (`_vrs_nm_perms`). Warn-level: GPU DPM not-`auto`, ZRAM state, live nftables NDP cross-check, ntsync runtime device state, iwd runtime state (opt-in), TSC demotion advisory. Advisory INFO (never fails): the 5 `_kb_*` subs. No THP, KSM, `ttm.*`, drirc, `radv_enable_unified_heap_on_apu`, or `iommu=pt` assert exists — do not verify them. NOTE: the GPU DPM assert is now `auto`; the IOMMU assert is now `off`/0-groups.

### Security delta (ordered)

1. **UMIP off** (`clearcpuid=514`) — descriptor-table base leak, kernel tainted; headline open reduction.
2. **AMD-Vi fully disabled** (`amd_iommu=off`, INVERTED from `iommu=pt`) — no DMA isolation/remapping; any DMA-capable device (USB4/Thunderbolt, NVMe, NIC) can in principle DMA over system RAM unmediated. Quantify the exposure and confirm it is an accepted trade for a no-passthrough single-user desktop. This is now an **open** reduction, not the prior "closed/restored" item.
3. **split_lock_detect=off** — a misbehaving app can degrade the system.
4. **Plaintext DNS** (`DNSOverTLS=no`, `DNSSEC=allow-downgrade`) reverting the CachyOS DoH default — DNS observable and spoofable on-path.
5. **Optional inbound remote-play ports** (`RY_REMOTE_PLAY_PORTS`, default OFF) — when enabled, opens Sunshine/Steam stream ports; quantify the exposure of the opt-in.
6. **Firewall default-deny-inbound ships** (nftables; no inbound IPv4 ping) — net positive.

## Investigation (§1–§12 ordered by installer phase; §13 = candidate enhancements)

### 1. Platform baseline and version floors

Current: **hard kernel floor KERNEL_MIN 6.18** (preflight `_ir_validate_kernel_floor`, override `RY_INSTALL_SKIP_KERNEL_FLOOR_CHECK=1`, fail-closed on unreadable `uname -r`) — the floor is anchored to **gfx1151 GPU/ROCm stability** (avoid < 6.18.4; 6.18.6+ recommended), NOT the RTL8127 NIC (whose true floor is 6.17); CPU gate `Ryzen AI Max` (override `RY_INSTALL_SKIP_HARDWARE_CHECK=1`, fail-closed on unreadable model); soft Mesa < 26.0 warn; **linux-firmware advisory** (hard-warn `20251125*` = bad GC 11.5.1 MES 0x83; soft-warn < `20260110` = the revert-to-MES-0x80 release).

- **6.18 floor — RESOLVED (Round 2, PRIMARY-VERIFIED).** The two r8169 commits do NOT gate 6.18: `f24f7b2f3af9` ("r8169: add support for RTL8127A", ChunHao Lin) landed mainline **6.16**; `ae1737e7339b` ("r8169: fix RTL8127 hang on suspend/shutdown", Heiner Kallweit, carries `Cc: stable`, `Fixes: f24f7b2f3af9`) landed mainline **6.18** and was backported to **6.17-stable** (carried as PATCH 6.17 096/146). So the networking floor is **6.17**. The hard 6.18 floor is instead justified by **gfx1151 stability**: kernels < **6.18.4** carry a documented gfx1151 stability bug and **6.18.6+** is recommended (Strix Halo toolbox maintainers; AMD ROCm RDNA3.5 sets a separate lower HWE floor of 6.17.0-19.19). KERNEL_MIN=6.18 is correct; the rigorous gfx1151 floor is 6.18.6. *(Optional sharpening: tighten KERNEL_MIN to 6.18.6 if the prompt wants the rigorous floor rather than the integer floor matching the script.)*
- **linux-firmware anchors — RESOLVED (Round 2, commit-pinned).** `20251125` carries the regressed GC 11.5.1 microcode (gfx1151 MES → **0x83**, commit `9643cbf2…`), confirmed to break ROCm on gfx1151. `20260110` is NOT a new fix — it is the **revert** to stable MES 0x80 (`c092c7487e…`). Known-good: **20251111** (MES 0x80) or **20260110+**; a corrected GC 11.5.1 re-add (`a0f0e52138…`, gfx1150/1151) exists but its release tag is unconfirmed (still-open item — keep the soft-warn pinned to 20251125). Latest release 20260624. There is no separate "MES 0x86" blob — that label in earlier prompt text is stale; the relevant versions are 0x83 (bad) vs 0x80 (good).
- Confirm the soft Mesa 26.0 floor matches current RADV guidance; enumerate open gfx1151 RADV issues and state whether 26.0 is still the right threshold.
- Confirm gfx1151 reports `uma:1` natively, so the removed drirc override is genuinely redundant on current Mesa.
- Sources: wiki.cachyos.org, docs.kernel.org gpu/amdgpu, gitlab.freedesktop.org/mesa (gfx1151), git.kernel.org linux-firmware + r8169.

### 2. Packages

PKGS_ADD (17): nvme-cli, cachyos-gaming-meta, cachyos-gaming-applications, lib32-mesa, mkinitcpio-firmware, fd, sd, dust, procs, bottom, htop, git-delta, lm_sensors, rtkit, realtime-privileges, ddcutil, nftables. PKGS_DEL (9, `-Rns`, rdep-aware): plymouth stack (plymouth, cachyos-plymouth-bootanimation, cachyos-plymouth-theme, breeze-plymouth, plymouth-kcm) + micro + cachyos-micro-settings + cachy-update + kdeconnect. AUR: none. Vulkan (chwd): vulkan-radeon, lib32-vulkan-radeon.

- Confirm cachyos-gaming-meta and -applications supply RADV/Proton/gamescope/MangoHud/GameMode, and that the meta pulls MangoHud (the profile also ships MangoHud.conf — confirm no conflict). **GameMode integration gap — RESOLVED (Round 2): KEEP the omission.** `gamemode`/`lib32-gamemode` are NOT in PKGS_ADD explicitly and the profile ships no `gamemode.ini`, no `gamemoderun` wrapper, and no GameMode env. Feral GameMode's primary effect is switching the CPUFreq governor to `performance` while a game runs ("There isn't any 'magical' performance tweaks … mainly about ensuring … the governor is set to performance") — but this profile already pins governor/EPP/DPM **profile-wide**, so GameMode's governor switch is redundant and its nice/ioprio hints are marginal. The static profile already covers what GameMode would do; KEEP the omission and document why (no `gamemode.ini` / `gamemoderun` needed). IMPACT Low · RISK Low.
- `rtkit`: confirm RealtimeKit is correct alongside realtime-privileges for PipeWire thread priority; is `rtkit-daemon.service` socket-activated (not in EXPECTED_SERVICES)?
- Confirm `lib32-mesa` is still needed alongside `lib32-vulkan-radeon`, or now redundant.
- Confirm plymouth, micro, cachy-update, kdeconnect removal has no dependency fallout (note `-Rns` skips + logs a package with an external dependant rather than cascading).
- Advisory one-liner (script no longer probes repo tier as of v7.74.0): state whether znver/x86-64-v4 (AVX-512) repos benefit this build over v3.
- Sources: wiki.cachyos.org, wiki.archlinux.org/Gaming + PipeWire + RealtimeKit.

### 3. Kernel cmdline (16)

`8250.nr_uarts=0 amd_iommu=off amd_pstate=active btusb.enable_autosuspend=n clearcpuid=514 fsck.mode=force fsck.repair=yes nowatchdog nvme_core.default_ps_max_latency_us=0 pcie_aspm.policy=performance processor.max_cstate=1 quiet split_lock_detect=off tsc=reliable usbcore.autosuspend=-1 zswap.enabled=0`

INVERTED since v7.75.1 — re-audit from scratch:

- **amd_iommu=off (was `iommu=pt`):** AMD-Vi is now fully off — there is no `iommu=pt`, no `amd_iommu=on`. **ROCm sub-question RESOLVED (Round 2): `amd_iommu=off` does NOT break ROCm.** `rocminfo` reports `IOMMU Support: None` on gfx1151, and the Strix Halo LLM-toolbox community *recommends* `amd_iommu=off` for the 128 GB unified pool (lower latency, no SVM/large-allocation penalty for llama.cpp/ROCm). So this is NOT a compute regression — do not escalate to FIX or a compute-vs-gaming opt-in. Still re-evaluate the remaining parts at audit time: (a) the gaming/compute latency win of disabling vs `iommu=pt` translation-bypass on Zen 5 (likely marginal); (b) the DMA-isolation loss (USB4/Thunderbolt, NVMe, NIC unmediated) — the security cost, quantified in §10; (c) confirm the README opt-back-in (`amd_iommu=on iommu=pt`, re-run, `_vrkm_iommu` asserts groups populated) is the correct VFIO/SR-IOV path. Net: accepted single-user-desktop trade-off, not a free latency win and not a compute regression.

Carry-forward NEW params (still first-pass — keep validating):

- **processor.max_cstate=1:** capping C-states trades idle power for wake-latency/jitter. Confirm beneficial on Strix Halo for frametime consistency; quantify idle-power/thermal cost on a 140 W package; confirm it does not fight amd_pstate=active or starve boost headroom. Is `1` the right cap?
- **btusb.enable_autosuspend=n:** confirm disabling BT USB autosuspend is the correct fix for MT7925/BT 5.4 reconnect/stutter (alongside the mt7925e ASPM drop-in §8/§11). Does it overlap with `usbcore.autosuspend=-1`, making one redundant?
- **fsck.mode=force + fsck.repair=yes:** forcing fsck every boot on dual NVMe ext4 — confirm safe and intended (boot-time cost, behavior on a dirty/large filesystem, interaction with systemd fsck units and the `fsck` mkinitcpio hook). Auto-repair without prompt: confirm no data-loss risk on ext4. Is "force every boot" the right durability posture, or should it be periodic?

Carry-forward (confirm still current):

- **clearcpuid=514 (UMIP off):** benefit (avoiding umip_printk stutter under anti-cheat/Wine) vs security cost (descriptor-table leak, taint). Confirm whether current Proton/EAC/BattlEye actually trip UMIP emulation on Zen 5. Recommend dropping if no stutter observed. (`_ry_check_umip_disabled` runs in preflight — confirm what it asserts.)
- **amd_pstate=active:** confirm recommended on Zen 5; interaction with powersave governor + balance_performance EPP (§6).
- **split_lock_detect=off:** perf vs stability; current default; blast radius.
- No `preempt=` pinned: `_vrk_cmdline` reads the runtime model from dmesg (`Dynamic Preempt: <mode>`) and only INFOs it; `_ok` only when the string contains `full`. **RESOLVED (Round 2): KEEP-omitted — pinning is redundant.** The CachyOS default desktop kernel ships `CONFIG_PREEMPT=y` + `CONFIG_PREEMPT_DYNAMIC=y` with the boot default already **full** (only the `-server` variant defaults to lazy), so `preempt=full` on the cmdline is a no-op on the default kernel. The INFO-only check is correct; do not add `preempt=full` to KERNEL_PARAMS. (If a CachyOS update ever flips the default to lazy/voluntary, revisit.)
- Zero amdgpu/ttm module params: confirm hands-off GPU-param posture is correct (`_vrkm_amdgpu` is hex-aware but no-ops when KERNEL_PARAMS carries no `amdgpu.*`).
- Validate the rest: tsc=reliable, nowatchdog, 8250.nr_uarts=0, usbcore.autosuspend=-1, nvme_core.default_ps_max_latency_us=0, pcie_aspm.policy=performance, zswap.enabled=0.
- Sources: docs.kernel.org kernel-parameters + pm/amd-pstate + x86 UMIP + admin-guide (processor.max_cstate, fsck) + IOMMU/AMD-Vi, wiki.archlinux.org/AMDGPU + IOMMU + fsck, amd.com ROCm (IOMMU/SVM requirements).

### 4. Bootloader and initramfs

loader.conf: default @saved, timeout 0, console-mode keep, editor no. sdboot-manage: DEFAULT_ENTRY manual, OVERWRITE/REMOVE_EXISTING/REMOVE_OBSOLETE yes, LINUX_FALLBACK_OPTIONS "quiet". mkinitcpio: MODULES=(amdgpu), HOOKS (11) = base systemd autodetect microcode modconf kms keyboard sd-vconsole block filesystems fsck, **COMPRESSION zstd, COMPRESSION_OPTIONS=(-1 -T0)**. `mkinitcpio.conf` is pre-deployed in Phase 2 so the `pacman -Syu` triggers exactly one initramfs rebuild.

- Verify HOOKS order with the systemd hook (microcode/kms/sd-vconsole/block placement); confirm amdgpu + kms is recommended early-KMS for this GPU. With `fsck.mode=force` on the cmdline (§3), confirm the `fsck` hook + fsck.repair handshake produces no boot prompt or hang.
- **COMPRESSION_OPTIONS=(-1 -T0):** the profile pins zstd level **-1** (negative = fastest/lowest-ratio, not the zstd default of 3) with `-T0` (all threads). On dual NVMe this trades a larger initramfs for the fastest decompression/boot. Quantify: (a) is the decompression-time delta between `-1` and default-3 measurable at boot on this NVMe (likely sub-100ms), and (b) does the larger `-1` image risk the 200/500 MB ESP `BOOT_SPACE_*` gates (§E) with multiple kernels + fallback resident? If the size cost threatens the ESP budget but the boot-time win is negligible, TUNE toward default-3 or `-T0` alone. Confirm `-T0` actually parallelizes at this level (zstd ignores threads below a size threshold).
- timeout 0 + DEFAULT_ENTRY manual + REMOVE_EXISTING=yes wipes foreign BLS entries (EFI loaders untouched); confirm current and intended.
- Confirm sdboot-manage is current and maintained vs kernel-install/UKI (UKI out of scope).
- Sources: wiki.archlinux.org/Mkinitcpio + systemd-boot, sdboot-manage upstream.

### 5. GPU / Vulkan / gaming

No drirc shipped (gfx1151 reports uma:1 natively). No ttm/modprobe params (kernel auto-sizes GTT). ENV_VARS (11): AMD_VULKAN_ICD=RADV, DXVK_LOG_LEVEL=none, DXVK_LOG_PATH=none, MANGOHUD=1, MESA_SHADER_CACHE_MAX_SIZE=16G, PROTON_ENABLE_WAYLAND=1, PROTON_FSR4_RDNA3_UPGRADE=1, PROTON_LOCAL_SHADER_CACHE=1, VKD3D_DEBUG=none, VKD3D_SHADER_DEBUG=none, WINEDEBUG=-all.

- **ntsync (DE-MANAGED — assert-only, autoload conf dropped v7.76.1):** the profile no longer ships any `modules-load.d` autoload; ntsync is asserted in preflight + verify only, and README states mainline ≥ 6.14 with a per-title opt-out `PROTON_NO_NTSYNC=1`. Confirm: (a) ntsync is the current Wine-sync mechanism vs esync/fsync; (b) the CachyOS kernel ships `CONFIG_NTSYNC=y` (builtin) so `/dev/ntsync` exists without any autoload — i.e. dropping the conf is correct, not a regression; (c) the `loaded_nodev` state (module loaded, `/dev/ntsync` absent) is a real-enough failure mode to keep the explicit `_ntsync_state` handling even though nothing is deployed; (d) Proton consumes `/dev/ntsync` and improves frametimes on 16C/32T; (e) `PROTON_NO_NTSYNC=1` is the current correct per-title escape and not stale. If the CachyOS kernel does NOT build ntsync in, flag that assert-only (no autoload) leaves `/dev/ntsync` absent — a real gap. Note the README Tuning note cites ntsync mainline ≥ 6.14 while `KERNEL_MIN` is 6.18 — confirm the 6.18 floor (set for RTL8127, §1) already satisfies ntsync, so no separate ntsync floor is needed.
- **PROTON_FSR4_RDNA3_UPGRADE=1:** confirm current Proton / Proton-CachyOS actually consume this variable to upgrade FSR3.1→FSR4 on RDNA 3.5 (gfx1151), the minimum Proton-CachyOS version, and whether it is a no-op or harmful on titles without FSR. README claims FSR4 reached RDNA3/3.5 via Proton-CachyOS — verify against current releases. If unverified upstream, flag FIX-to-remove; if verified, KEEP and cite.
- **RADV unified heap (drirc removed):** confirm current RADV on gfx1151 reports uma:1 and treats the heap as unified without the override. If not, flag a regression.
- **GTT sizing (ttm removed):** confirm the installed kernel auto-sizes GTT sensibly (~62 GiB ceiling). README directs >~62 GiB single allocations (ROCm/llama.cpp) to the **BIOS UMA carveout** (up to 96 GB), not deprecated `amdgpu.gttsize`, and verifies via `cat /sys/module/ttm/parameters/pages_limit`. Confirm this is the current correct mechanism and that removing the cap does not under-provision compute. **Cross-check with §3 — RESOLVED (Round 2):** `amd_iommu=off` does NOT change the usable GTT/SVM ceiling or break large ROCm allocations (gfx1151 reports `IOMMU Support: None`; the toolbox community runs `amd_iommu=off` with the full unified pool). The carveout vs GTT-auto question is independent of the IOMMU setting.
- PROTON_ENABLE_WAYLAND=1: maturity and fallback on current Proton.
- AMD_VULKAN_ICD=RADV: confirm it reliably forces RADV vs VK_DRIVER_FILES.
- MESA_SHADER_CACHE_MAX_SIZE=16G + PROTON_LOCAL_SHADER_CACHE=1: confirm no conflict, sane sizing.
- MANGOHUD=1 global enable: confirm low-overhead vs per-launch, clean with gamescope/GameMode.
- Sources: docs.mesa3d.org (RADV, APU heap), gitlab.freedesktop.org/mesa + drm/amd (GTT auto-sizing kernel version), github Proton/Proton-CachyOS (FSR4, ntsync), amd.com ROCm, wiki.cachyos.org.

### 6. CPU performance and power

amd_pstate=active; governor **powersave** (honors EPP under active mode; sourced from `/etc/default/cpupower-service.conf` `GOVERNOR`); EPP **balance_performance** via udev (add|change, re-asserts after AC/DC); **GPU clock-floor GPU_DPM_LEVEL=auto** (parameterized, add-only udev rule, single-digit `card[0-9]` match). Masked: power-profiles-daemon, ananicy-cpp, modemmanager. Installed: realtime-privileges, rtkit.

- **governor=powersave + EPP=balance_performance (LIVE, special case):** **CONFIRMED (Round 2)** — under `amd_pstate=active` only the `powersave`/`performance` pseudo-governors exist; `powersave` honors the EPP hint (dynamic scaling), `performance` pins max and ignores EPP. The active + powersave + balance_performance triple IS the documented EPP-honoring max-perf config on Zen 5. Do not flag powersave; do not flip the governor. The `balance_performance` → `performance` EPP question (does mid-bias leave 1%-lows on the table?) is **UNCERTAIN** — no gfx1151/Zen-5 gaming frametime comparison between the two EPP values exists (available EPP data is server/EPYC perf-per-watt). Mark the EPP=performance opt-in UNCERTAIN rather than recommending it; if a Zen-5 gaming EPP comparison appears, resolve then (the change would be the udev EPP ATTR only; `CPUPOWER_GOVERNOR` stays powersave). IMPACT UNCERTAIN · RISK Low.
- **GPU_DPM_LEVEL=auto:** the profile does not pin SCLK. Confirm `auto` is the right default — (a) does leaving DPM at `auto` cost frametime/1%-lows on gfx1151 vs pinning `high`, or was pinning only burning idle power? (b) on a shared 140 W package with EPP=balance_performance, does `auto` correctly let firmware arbitrate CPU↔GPU power? (c) any title class where `high` still wins enough to warrant a documented opt-in? Note the udev rule is `add`-only (one-shot at enumeration): if a reviewer recommends `high`, the rule would NOT re-assert after a GPU reset — flag that as a reason `auto` is the more robust default.
- Confirm udev add|change EPP pinning is robust vs one-shot; prefcore=enabled + boost=1 correct on Strix Halo.
- Mask power-profiles-daemon: confirm no needed platform_profile path lost; does CachyOS expect ppd? Consider tuned.
- Mask ananicy-cpp: confirm net win for gaming.
- Mask modemmanager: confirm no cellular HW, masking loss-free (the masked-ModemManager D-Bus log line is flagged benign by `_kb_modemmanager_masked` §12 — confirm cosmetic).
- Sources: docs.kernel.org pm/amd-pstate (active + EPP + governor interaction), wiki.archlinux.org/CPU_frequency_scaling + AMDGPU (power_dpm_force_performance_level), freedesktop ppd.

### 7. Memory and storage

zswap.enabled=0; NVMe scheduler none (udev `99-ry-perf.rules`, sorts after vendor 60-ioschedulers.rules); **SYSCTL_VALUES (9):** net.core.default_qdisc=fq, net.core.netdev_budget=600, net.core.netdev_budget_usecs=5000, net.ipv4.tcp_congestion_control=bbr, net.ipv4.tcp_notsent_lowat=16384, net.ipv4.tcp_slow_start_after_idle=0, vm.compaction_proactiveness=0, vm.max_map_count=2147483642, vm.swappiness=150 (priority 95 `95-ry-overrides.conf` after vendor 70-cachyos-settings.conf); fstab ext4 noatime,lazytime,commit=10; THP/KSM/systemd-oomd left to CachyOS. `vm.page-cluster` and `vm.vfs_cache_pressure` remain DROPPED (vendor duplicates).

- **Confirm the vendor-duplicate drop is still a no-op:** confirm CachyOS `70-cachyos-settings.conf` (or another vendor sysctl.d file) actually sets `vm.page-cluster=0` and `vm.vfs_cache_pressure=50` — i.e. the profile's removal leaves the same effective values. If the vendor default differs, the removal is a SILENT behavior change for zram readahead and cache reclaim — flag it. List exact vendor values.
- **zram pair:** confirm the running kernel accepts swappiness > 100; is 150 gratuitous on 128 GB RAM, or does it help large-alloc/LLM reclaim? The priority-95 file now overrides just swappiness/compaction/max_map_count from vendor 70-cachyos — confirm justified.
- zswap.enabled=0: confirm CachyOS uses zram (not zswap) by default — no double-compression conflict.
- NVMe scheduler none: confirm best practice vs mq-deadline/kyber on this kernel. **`nr_requests` and `read_ahead_kb` are NOT currently set by the profile** (only `scheduler=none` is deployed via `99-ry-perf.rules`). Determine whether tuning either materially helps game-load/asset-streaming or large-sequential LLM-weight reads on this NVMe, and if so propose concrete values as a udev ATTR addition (e.g. `read_ahead_kb` raised for sequential throughput); else state explicitly that kernel defaults are optimal and no knob should be added. Confirm `99-ry-perf.rules` sorts after vendor `60-ioschedulers.rules` (last-matching ATTR assignment wins).
- fstab noatime,lazytime,commit=10: confirm noatime and lazytime coexist; weigh commit=10 durability (esp. with `fsck.mode=force` every boot §3); confirm fstrim.timer over continuous discard.
- vm.max_map_count near 2^31: confirm appropriate for Proton/anti-cheat. compaction_proactiveness=0: confirm right for gaming + large unified allocs.
- Confirm not enabling systemd-oomd (kernel OOM + zram) is right on 128 GB.
- Sources: docs.kernel.org (block, sysctl/vm), wiki.archlinux.org/Zram + SSD + Ext4, wiki.cachyos.org (zram + sysctl defaults).

### 8. Network and latency

sysctl net: default_qdisc=fq, netdev_budget=600, netdev_budget_usecs=5000, tcp_congestion_control=bbr, tcp_notsent_lowat=16384, tcp_slow_start_after_idle=0. NM: wifi.backend=wpa_supplicant (NM default; iwd opt-in via NM_WIFI_BACKEND), wifi.powersave=2 (off), logging WARN. `/etc/modprobe.d/60-ry-mt7925e.conf` → `options mt7925e disable_aspm=1`. resolved: MulticastDNS=no, LLMNR=no, DNSOverTLS=no, DNSSEC=allow-downgrade (plaintext; diverges from CachyOS DoH default). regdom: COUNTRY=US fixed (`/etc/iw-regdomain`). Masked: NetworkManager-wait-online, modemmanager. Enabled: NetworkManager.

- **mt7925e disable_aspm=1:** confirm disabling PCIe ASPM on MT7925 is the current correct mitigation for coredump/BT-reconnect/assoc-fail, and whether it is still needed or an upstream mt76 fix has landed (→ if landed, prefer a kernel/firmware floor and flag the drop-in as removable). Confirm it does not fight `pcie_aspm.policy=performance` (§3) — are both saying "no ASPM" on this device, making one redundant?
- **NM backend wpa_supplicant:** verify current Arch/CachyOS guidance for MT7925/mt76 — is wpa_supplicant the more stable backend today, or has iwd matured to parity? Confirm wifi.powersave=2 alone fully disables the mt76 software power-save latency issue under wpa_supplicant, and that backend + ASPM drop-in + btusb.enable_autosuspend=n (§3) together close the MT7925 stability gap. Confirm no dangling iwd reference (`iwd/main.conf` removed v7.64.0).
- bbr + fq: confirm still recommended; clarify BBRv3 status in current kernels.
- Dual 10 GbE: validate netdev_budget/usecs for 10G; assess tcp_rmem/wmem or NIC ring tuning for line-rate (RTL8127, §11).
- tcp_notsent_lowat=16384, tcp_slow_start_after_idle=0: confirm rationale holds.
- mDNS off (consistent with nftables drop §10); plaintext DNS reverting DoH — confirm CachyOS ships an encrypted-DNS default this overrides; flag the privacy reduction (§10).
- regdom US fixed: confirm correct max TX power and channel set for MT7925 on current cfg80211/wireless-regdb. Flag if 6 GHz Wi-Fi 7 needs a specific AFC posture beyond a country code; note non-US deployers must hand-edit COUNTRY. (README notes the `3 dBm` TX-power readout is cosmetic — correct power applied; confirm.)
- Sources: docs.kernel.org/networking (bbr, fq, tcp), wiki.archlinux.org/Sysctl + NetworkManager + Wireless/MediaTek, git.kernel.org wireless-regdb + mt76.

### 9. systemd units, time-sync

Mask (10): ananicy-cpp, power-profiles-daemon, NetworkManager-wait-online, ufw, modemmanager, sleep/suspend/hibernate/hybrid-sleep/suspend-then-hibernate targets. Enable (5): fstrim.timer, NetworkManager, cpupower, nftables, bluetooth. Not enabled: systemd-oomd (intentional), NetworkManager-dispatcher (socket-activated), rtkit-daemon (socket-activated). iwd.service untouched (opt-in). ufw flushed after nftables live. **NEW: `_ry_rtc_writeback` (`hwclock --systohc --utc`) at both sync-confirmed paths.**

- For each mask, confirm safe and beneficial on CachyOS: ananicy-cpp + ppd (§6); modemmanager (no cellular HW); sleep/suspend masked = no suspend at all (matches an always-on mini-PC).
- **RTC write-back (`_ry_rtc_writeback`, NEW):** confirm `hwclock --systohc --utc` after NTP sync is safe and that the `RTCInLocalTZ=yes` guard (defer to systemd) is the correct branch. Confirm the rationale — a skewed RTC poisoning systemd-timer persistence stamps (e.g. `Persistent=true` timers) — is real, and that a UTC-only direct write does not conflict with `systemd-timesyncd` ownership of the RTC.
- bluetooth.service enabled (with main.conf §12): confirm AutoEnable=true posture and BlueZ key currency.
- Confirm nftables is enabled as the firewall and that ufw-flush-then-mask leaves no unfirewalled window (script skips the ufw mask if nft is not confirmed live — confirm this handoff is correct).
- fstrim.timer vs continuous discard; cpupower vs CachyOS freq management; confirm oomd stays disabled and dispatcher/rtkit stay socket-activated.
- logind Handle*Key=ignore incl LongPress: confirm intended, no lockout risk.
- Sources: man.archlinux.org (systemd.unit, logind.conf, hwclock, systemd-timesyncd), wiki.archlinux.org (Bluetooth, System time), wiki.cachyos.org.

### 10. Security and safety (cross-cutting)

nftables default-deny-inbound (ufw masked): input policy drop, ct established/related accept, lo accept, ct invalid drop, ICMPv6 NDP/PMTUD + echo-request accept, IPv4 ICMP diagnostics-only (no inbound ping), forward drop, output accept. `RY_REMOTE_PLAY_PORTS` gate (default false) appends `tcp dport { 47984, 47989, 48010, 27036 }` + `udp dport { 47998-48010, 27031-27036 }`. **amd_iommu=off (AMD-Vi DISABLED).** clearcpuid=514 (UMIP off). split_lock_detect=off.

- **amd_iommu=off (INVERTED — now the #2 open reduction, not a "closed" item):** quantify the DMA-isolation loss with AMD-Vi fully off (USB4/Thunderbolt, NVMe, NIC DMA unmediated). Confirm this is the intended posture for a no-passthrough single-user desktop and that the README escape hatch (`amd_iommu=on iommu=pt` for VFIO/SR-IOV) is correct. List it second in the security delta. Cross-link §3: the ROCm/SVM concern is **RESOLVED (Round 2)** — `amd_iommu=off` does not break ROCm on gfx1151 (`IOMMU Support: None`), so there is no compute-side escalation; the trade-off is purely security (DMA isolation) vs latency, and it is accepted.
- **RY_REMOTE_PLAY_PORTS port set:** validate the exact ports against current Sunshine/Moonlight + Steam Remote Play requirements. Confirm `47984/47989/48010` (Sunshine HTTPS/HTTP/RTSP), `27036` (Steam), and the UDP ranges are correct and complete — flag any missing port (e.g. Sunshine video/control/audio/mic UDP) or any stale one. Confirm default-OFF is right and that enabling appends cleanly.
- Validate the nftables shape is minimal-but-sufficient on a dual-10 GbE LAN.
- IPv4 ping dropped, IPv6 reachable: confirm the intended asymmetric LAN posture.
- ICMPv6 NDP/PMTUD accept must be present (static `_vss_nft` / `_vrsv_nft_assert_ndp` hard-fail if nd-neighbor-solicit is missing; live check warn-only). Treat the static rule as the gate.
- clearcpuid=514 is the headline open reduction; quantify exposure vs the umip_printk stutter prevented (§3); list first.
- split_lock_detect=off: quantify residual exposure.
- Produce the ordered security-delta subsection (above), with amd_iommu=off at position 2.
- Sources: wiki.archlinux.org (nftables, Security, IOMMU), docs.kernel.org (split lock, UMIP, AMD-Vi), github Sunshine/Moonlight (port reference).

### 11. Known issues and DKMS currency

MES page faults → the gfx1151 MES regression is a **firmware-version** issue, not a one-time fix: bad GC 11.5.1 microcode (MES **0x83**, commit `9643cbf2…`) shipped in `20251125`, reverted to stable MES 0x80 (`c092c7487e…`) in `20260110` — preflight hard-warns on `20251125*`, soft-warns < `20260110`. RTL8127 throughput + suspend/shutdown hang → resolved in-tree r8169 (PRIMARY-VERIFIED: `f24f7b2f3af9` support in **6.16**; `ae1737e7339b` suspend fix in **6.18**, backported **6.17-stable**); the KERNEL_MIN 6.18 floor is set for **gfx1151 stability**, not these NIC commits (NIC floor = 6.17); no DKMS. MT7925 panics/deauth/coredump → mitigated via `disable_aspm=1` drop-in (§8) + `btusb.enable_autosuspend=n` (§3) + wpa_supplicant; **upstream status improving, not fully closed** — mt7925e stability fixes landed **6.17+** (so the 6.18 floor already carries them), but instability is still reported on some platforms, so the drop-in stays defensive. Strix Halo ACP → **confirmed still open (Round 2)**: ACP70 has no upstream internal-mic ASoC machine driver / UCM profile as of mid-2026 (only ASUS SoundWire *speaker* quirks landed, ~kernel 7.0); internal mic undetected, flagged by `_kb_acp70_no_machine_driver` §12. Install pacman-only.

- **MES claims — RESOLVED (Round 2).** The relevant microcode versions are MES **0x83** (bad, `20251125`, commit `9643cbf2…`) vs **0x80** (good; `20251111` and the `20260110` revert `c092c7487e…`). There is no "MES 0x86" blob — that label was stale. Minimum kernel containing BOTH r8169 commits = **6.18** mainline (the suspend fix `ae1737e7339b` lands 6.18 / 6.17-stable), but this does not set the floor (gfx1151 does). Firmware hashes are secondary-corroborated (Launchpad #2129150 / Debian changelog / ROCm #5724); git.kernel.org blocked direct fetch.
- **MT7925 — RESOLVED (Round 2):** mt7925e stability fixes landed **6.17+**, already covered by the 6.18 floor; keep the `disable_aspm=1` drop-in as defensive (instability still reported on some platforms). wpa_supplicant vs iwd remains a wash — keep wpa_supplicant. Status: improving, not a closed gap.
- **ACP/internal mic — RESOLVED (Round 2): still open upstream.** No ACP70 internal-mic machine driver / UCM profile exists as of mid-2026 (alsa-ucm-conf has no acp70 profile; only ASUS TAS2783 *speaker* SoundWire quirks landed ~kernel 7.0). Nothing the profile can ship — document as a known-gap; do NOT recommend a kernel floor for it. Reporting the board model upstream remains the correct action.
- Recommend a kernel/firmware floor over DKMS for any landed fix; confirm any suggested DKMS still builds.
- Sources: gitlab.freedesktop.org/drm/amd, git.kernel.org linux-firmware + r8169 (`f24f7b2f3af9`, `ae1737e7339b`) + mt76, bugzilla.kernel.org, discuss.cachyos.org.

### 12. MangoHud, Bluetooth, and hygiene

**MangoHud.conf (17 active directives + 1 commented, 0600, fps/frametime first):** horizontal, legacy_layout=0, position=top-left, toggle_hud=Shift_R+F12, fps, frametime, frame_timing, gpu_stats, gpu_core_clock, gpu_temp, gpu_power, cpu_stats, **`# cpu_temp` (COMMENTED OUT v7.76.1; re-enable with `cpu_temp_sensor=k10temp`)**, cpu_mhz, vram, ram, font_size=20, text_outline, background_alpha=0.4. Still fully absent: fps_metrics. Enabled via MANGOHUD=1. bluetooth main.conf: FastConnectable=true, AutoEnable=true, ReconnectAttempts=3 (no explicit ReconnectIntervals — BlueZ default backoff). baloofilerc: Indexing-Enabled=false.

- **MangoHud `cpu_temp` (RESOLVED, Round 2):** the directive was added v7.76.0 then commented v7.76.1 pending per-host hwmon resolution. The correct re-enable key is **`cpu_temp_sensor=k10temp`** (NOT `cpu_custom_temp_sensor`; MangoHud #1825) — `k10temp`/Tctl is the correct Zen 5 CPU sensor. **CAVEAT (open bug):** enabling `cpu_temp` on Zen 5 currently zeroes the `cpu_power` readout (MangoHud #1794), so re-enabling both `cpu_temp` and the existing `cpu_power` together sacrifices CPU-power telemetry until that bug lands. Recommendation: re-enable `cpu_temp` + `cpu_temp_sensor=k10temp` (TUNE) only if CPU thermal matters more than CPU power on this host; otherwise keep commented until #1794 resolves. Verify the exact key against the installed MangoHud version's config reference before deploying.
- Confirm all 17 active directives are valid for current MangoHud (do not invent). With `gpu_power` present, confirm it populates from amdgpu sensors on gfx1151 under Wayland (the most decision-relevant readout on a 140 W shared-package APU). Confirm `frame_timing` (graph) vs `frametime` (numeric) are both current and not redundant. Confirm `toggle_hud=Shift_R+F12` is a valid current bind. Establish the real minimum MangoHud + kernel for the amdgpu sensors in use.
- Confirm gpu_temp/gpu_core_clock/vram/cpu_mhz populate from amdgpu sensors under Wayland on this iGPU; confirm vram + ram is the right unified-memory representation.
- Overhead: confirm near-zero, no conflict with the gamescope overlay or GameMode.
- **Bluetooth main.conf:** confirm BlueZ keys/sections current; ReconnectAttempts=3 + BlueZ default backoff sane for paired audio sinks; AutoEnable=true fixes adapter-off-at-boot; FastConnectable=true no meaningful downside on an always-on desktop. Cross-check `btusb.enable_autosuspend=n` (§3) — confirm kernel-level autosuspend-off and BlueZ-level reconnect policy are complementary, not redundant.
- baloofilerc Indexing-Enabled=false: confirm [Basic Settings]/Indexing-Enabled current for the installed Baloo.
- **Known-benign verify surface (`_kb_*`, 5 INFO subs):** confirm each is correctly characterized as benign on this exact hardware and none masks a real fault: (1) `_kb_modemmanager_masked` (masked ModemManager D-Bus noise), (2) `_kb_acp70_no_machine_driver` (ACP70 no ASoC machine driver, mic undetected), (3) `_kb_thunderbolt_nhi_unknown` (boltd unknown-NHI-PCI-id, USB4/TB UID-stability skipped), (4) `_kb_no_battery_backlight` (no battery/backlight sysfs), (5) `_kb_usb_mic_volume_curve` (USB-mic UAC volume-curve quirk). Validate each is genuinely cosmetic/expected, not a swallowed regression.
- **Naming:** every human-facing string reads "GTR9 Pro" but the internal PROFILE_NAME and function names carry `gtr_pro` (FIX, Low/Low) — do not change if it breaks function-name refs or log fields; cosmetic.
- Sources: github flightlessmango/MangoHud (config ref, version tags), wiki.archlinux.org/MangoHud + Bluetooth, man.archlinux.org bluetooth main.conf, KDE Baloo docs.

## 13. Candidate enhancements (absent knobs — gaming-first; ADD-opt-in vs KEEP-omitted)

This section is additive: each item is a knob the profile does NOT currently set, evaluated for whether it should be ADDED (as a default or a documented opt-in) or whether the omission is correct (KEEP). Anchor every call to gfx1151 / Zen 5 / RDNA 3.5 / current Mesa+Proton-CachyOS. Reserve ADD-as-default only for a clear, low-risk frametime/throughput win; otherwise propose an opt-in or KEEP-omitted with rationale. Never invent a flag — cite upstream or mark UNCERTAIN.

### 13a. Kernel cmdline candidates (none currently present)

- **`mitigations=off` — CALL: KEEP-omitted (Round 2).** Zen 5 (Ryzen AI 300 / 9000 series) is not affected by Inception/SRSO, and `mitigations=off` yields no measured benefit on Ryzen 9000-class parts outside synthetic kernel microbenchmarks. Residual default mitigations on Zen 5 are HW/microcode-handled: Spectre-V1 (usercopy/SWAPGS barriers + __user pointer sanitization), Spectre-V2 (enhanced/auto IBRS + always-on STIBP + conditional IBPB), SSB-V4 (prctl-controllable). No Proton/Wine gaming delta exists for this exact APU; the negligible-benefit conclusion is from a desktop 9950X, not the 395 — so KEEP-omitted, do NOT add to the security delta. *(Re-open as ADD-opt-in only if a published gfx1151 Proton frametime delta > ~2% appears.)* IMPACT Low · RISK Med (security).
- **`amdgpu.ppfeaturemask=0xffffffff` — CALL: KEEP-omitted (Round 2).** GPU undervolt/OC is NOT implemented for Strix Halo: `pp_power_profile_mode`, overdrive, and power-cap report "Not supported on the given system" on gfx1151 even with the full pp_feature mask applied (ROCm #5750), and the GPU cannot exceed the shared 140 W package cap regardless. The real power lever is **CPU** undervolt via `ryzenadj` (community reports ~−40 mV recovering throttle losses; −50 mV destabilized the desktop) — out of scope for a kernel cmdline param. Keep removed. IMPACT Low · RISK Med.
- **`preempt=full` — CALL: KEEP-omitted, redundant (Round 2).** The CachyOS default desktop kernel already ships `CONFIG_PREEMPT_DYNAMIC=y` with boot default = **full** (only `-server` defaults to lazy), so pinning is a no-op. See §3. IMPACT none · RISK none.
- **`nvme_core.io_timeout` / `pcie_port_pm=off` — CALL: KEEP-omitted.** Redundant alongside `nvme_core.default_ps_max_latency_us=0` + `pcie_aspm.policy=performance`; no concrete stutter/latency case. IMPACT Low · RISK Low.

### 13b. RADV / Mesa env candidates (ENV_VARS currently has no RADV_* tuning)

- **`RADV_PERFTEST` — CALL: KEEP-omitted (gpl/sam) / UNCERTAIN (nggc) (Round 2).** `gpl` is default-on since Mesa 23.1 (no longer a PERFTEST token; disable via `RADV_DEBUG=nogpl`), and `sam` is auto-enabled by the driver when all VRAM is CPU-visible (the APU case) — both KEEP-omitted. Current token set: cswave32, dccmsaa, dmashaders, gewave32, localbos, nggc, nircache, nogttspill, nosam, pswave32, rtcps, rtwave64, sam. None has a published RDNA3.5/gfx1151 measured win; `nggc` (NGG culling) is plausibly relevant but has **no gfx1151 benchmark** → mark **UNCERTAIN**, do not ADD on a guess. `rtwave64` hurts RDNA2; ignore. IMPACT Low · RISK Low.
- **`RADV_DEBUG=novrsflatshading` / correctness toggles — CALL: KEEP-omitted.** Only if a known gfx1151 rendering bug requires one — none currently identified. Flag any open RADV gfx1151 issue that a toggle works around at audit time.
- **`MESA_VK_WSI_PRESENT_MODE` / `vblank_mode` — CALL: KEEP-omitted (per-game).** Forcing a present mode (e.g. mailbox for low-latency uncapped) belongs per-title, not host-wide. Document the per-game pattern.
- **`mesa_glthread=true` — CALL: KEEP-omitted.** OpenGL-threading helps only native/older GL titles; most Proton games are Vulkan via DXVK/VKD3D. Not worth a global env. IMPACT Low · RISK Low.

### 13c. DXVK / VKD3D-Proton candidates

- **DXVK config (`DXVK_CONFIG` / `dxvk.conf`) — CALL: KEEP-omitted (auto optimal) (Round 2).** `dxvk.enableGraphicsPipelineLibrary=Auto` is default-on in current DXVK (enables GPL when supported, compiling optimized pipelines in the background) and `dxvk.numCompilerThreads=0` already auto-detects all cores — pinning gives no measured stutter improvement on a 16C/32T CPU. Legacy `DXVK_ASYNC` is unsupported and superseded by GPL (`gplAsyncCache` was removed in DXVK 2.7) — do NOT recommend the old async patch. IMPACT Low · RISK Low.
- **`PROTON_ENABLE_NGX_UPDATER` / upscaler envs — CALL: KEEP-omitted beyond §5.** Beyond `PROTON_FSR4_RDNA3_UPGRADE=1` (§5, confirmed live; on RDNA3.5 also needs `DXIL_SPIRV_CONFIG=wmma_rdna3_workaround` to avoid broken visuals), no host-wide upscaler/frame-gen env is frametime-relevant enough to globalize. Keep upscaler scope minimal/per-title.
- **`VKD3D_CONFIG` — CALL: KEEP-omitted (per-game).** DX12 toggles like `dxr` (ray tracing) belong per-title, not as a global default on gfx1151. IMPACT Low · RISK Low.

### 13d. Firmware / platform (verify-only; profile cannot set, but should check)

- **Resizable BAR / Smart Access Memory — CALL: verify-only, auto-on (Round 2).** On the Strix Halo APU all VRAM is CPU-visible (rocminfo shows the full unified pool), so RADV auto-enables sam optimizations — no need to force `RADV_PERFTEST=sam`. Optional advisory `_kb_*`-style INFO for ReBAR-off via rocminfo pool sizes / `lspci -vv` BAR sizes / amdgpu dmesg. Low priority (ReBAR-off is unlikely on this platform).
- **BIOS UMA carveout vs GTT — gaming verdict KEEP-omitted.** For gaming (not ROCm), the default GTT ceiling (~62 GiB) never bottlenecks a game's VRAM working set on the 128 GB pool — no carveout change needed for gaming. The README carveout note is compute-oriented; gaming needs nothing.

### 13e. Scheduler / memory (most already covered in §6–§7; only the gaps)

- **`read_ahead_kb` / `nr_requests` — CALL: KEEP-omitted, defaults optimal (Round 2).** Under the `none` scheduler, smaller read-ahead (128 KB) is the guidance for NVMe random I/O; defaults are appropriate. Game load is CPU-decompression/shader-bound, not sequential-throughput bound, and no game-load or LLM-weight-load benchmark shows benefit from raising either. State kernel defaults are optimal; add no knob. IMPACT Low · RISK Low.
- **`vm.max_map_count` — CALL: KEEP (sufficient) (Round 2).** 2147483642 (= MAX_INT−5, the SteamOS value) satisfies all current Proton/anti-cheat requirements; no title is known to need exactly 2^31−1. No change. (Arch default is now 1048576; the profile's raised value is the established max-compat value.)
- **CPU affinity / isolation (`isolcpus`, `nohz_full`, `rcu_nocbs`) — CALL: KEEP-omitted (wrong here).** Core isolation on a 16C/32T gaming desktop hurts: it removes cores from the scheduler and breaks GameMode/Proton thread placement. Isolation targets fixed-workload RT systems, not a desktop where the game wants every core. IMPACT Low · RISK Med (if added).

For each 13a–13e item: ITEM · PRESENT?(no) · CALL(ADD-default / ADD-opt-in / KEEP-omitted) · IMPACT · RISK · EVIDENCE. Bias toward KEEP-omitted unless the gaming win is concrete and low-risk; the profile is intentionally lean.

---

## Scope and non-goals

- Recommendations only — do not emit a modified script.
- Out of scope: dotfiles, shells, editors, secrets, backups, multi-user, non-CachyOS, laptops, UKI.
- Per-game Proton tuning is secondary; prioritize system-wide config.
- **Reinstatement rule.** Items deliberately removed/disabled (do not recommend reinstating unless current upstream directly contradicts the rationale — then flag, not FIX): `amdgpu.ppfeaturemask` (re-evaluate per §13a as an undervolt/OC opt-in, not a silent default), `--country` flag, TTM/GTT cap, RADV drirc, MangoHud `fps_metrics`, `vm.page-cluster`/`vm.vfs_cache_pressure` (vendor-provided), ntsync `modules-load.d` autoload conf (now assert-only). Live config to evaluate as KEEP-or-FIX-to-remove (not protected): PROTON_FSR4_RDNA3_UPGRADE, MangoHud gpu_power/text_outline/toggle_hud.
- **IOMMU special case (INVERTED):** the profile now ships `amd_iommu=off` (AMD-Vi fully disabled) — `iommu=pt` is gone. Do NOT recommend re-adding `iommu=pt`/`amd_iommu=on` as a default unless ROCm/compute on gfx1151 is proven to require the IOMMU, OR a DMA-isolation requirement is established for this single-user desktop. The VFIO/SR-IOV opt-in (`amd_iommu=on iommu=pt`) is the documented per-user override, not the profile default.
- **Governor/EPP special case:** `powersave` + `balance_performance` is the EPP-honoring config under `amd_pstate=active` — do not flag `powersave` as a regression without proving the `performance` governor would override the EPP hint.
- **GPU_DPM_LEVEL special case:** `auto` is deliberate (stop pinning SCLK and stealing Zen 5 boost). Do not flag `auto` as a GPU-perf regression without proving `high` materially improves frametime/1%-lows on gfx1151 without costing CPU boost on the shared 140 W package.

---

# Deep-pass appendix (exact generated bodies + full verify surface)

The §1–§13 investigation is value-level (§1–§12 = current config; §13 = absent-knob candidates). This appendix is artifact-level: the exact strings the script writes, the complete verify subsystem, and the install-phase model. Validate the **rendered file content**, not a paraphrase. Every block below is quoted from the generator functions in `ry-install.fish` v7.77.1 (UUIDs/joins resolved at runtime).

## A. Install-phase model (`_RY_PHASE_NAMES`)

Six ordered phases; recommendations must respect this sequence:

```
1 Preflight     _install_preflight     — _ir_* gates (counts, keys, kernel floor, post-hooks, root UUID); umip check; mesa + linux-firmware advisories
2 Packages      _install_packages      — mkinitcpio.conf pre-deployed → pacman -Syu PKGS_ADD (one initramfs rebuild); chwd Vulkan
3 Configuration _install_system_files  — render+deploy all 18 managed files (atomic tmp+rename)
4 Services      _install_configure_services — fstab rewrite + resolved + PKGS_DEL (-Rns) + mask (nft-first, then ufw flush) + iwd handoff + enable + regdom + RTC write-back
5 Boot          _install_rebuild_boot  — taint-gate → mkinitcpio -P + sdboot-manage gen/update (gated on boot-critical writes)
6 Finalize      _install_finalize      — user daemon-reload + paccache + NetworkManager restart
```

- Confirm the **firewall handoff lives in Phase 4** (nftables made live *before* ufw is flushed/masked) and that boot-critical regeneration (Phase 5) only fires when one of `_RY_BOOT_CRITICAL_DSTS` changed. Flag any recommendation that would move a cmdline/mkinitcpio change outside the Phase-5 gate.
- `_RY_BOOT_CRITICAL_DSTS` (4): `/boot/loader/loader.conf`, `/etc/kernel/cmdline`, `/etc/sdboot-manage.conf`, `/etc/mkinitcpio.conf`. `_RY_BACKUP_TARGETS` (2, `.ry.bak`): `/boot/loader/loader.conf`, `/etc/mkinitcpio.conf` (plus fstab during its rewrite). Confirm the backup set is sufficient (note `/etc/kernel/cmdline` is regenerable from `KERNEL_PARAMS`).
- `_RY_POST_HOOKS` (18 tags): baloo, bluetooth, boot (×3: loader.conf/sdboot.conf/mkinitcpio.conf), cmdline, cpupower, envd, logind, mangohud, modprobe, nft, nm, nmdispatch, regdom, resolved, sysctl, udev. `_ir_validate_post_hooks` refuses deploy if any tag lacks a handler. Confirm `--install-file <path>` of any single managed file triggers the correct reload and that modprobe/udev/cmdline handlers correctly defer to reboot.

## B. Exact rendered file bodies (validate content, not summary)

### B1. `/etc/kernel/cmdline` + `/etc/sdboot-manage.conf`

```
rw root=UUID=<_ROOT_UUID> 8250.nr_uarts=0 amd_iommu=off amd_pstate=active btusb.enable_autosuspend=n clearcpuid=514 fsck.mode=force fsck.repair=yes nowatchdog nvme_core.default_ps_max_latency_us=0 pcie_aspm.policy=performance processor.max_cstate=1 quiet split_lock_detect=off tsc=reliable usbcore.autosuspend=-1 zswap.enabled=0
```
sdboot-manage.conf: `LINUX_OPTIONS` = same KERNEL_PARAMS join; `LINUX_FALLBACK_OPTIONS="quiet"`; `DEFAULT_ENTRY="manual"`; `REMOVE_EXISTING="yes"`; `OVERWRITE_EXISTING="yes"`; `REMOVE_OBSOLETE="yes"`.

- **`amd_iommu=off` now ships in BOTH bootloader-management paths.** Confirm both `/etc/kernel/cmdline` (kernel-install) and `sdboot-manage.conf` `LINUX_OPTIONS` (sdboot-manage) are not simultaneously active in a conflicting way; state which CachyOS actually drives and whether maintaining both is redundant or a divergence risk.
- `LINUX_FALLBACK_OPTIONS="quiet"` strips ALL params from the fallback entry. Confirm a fallback boot with none of `amd_pstate`/`fsck.*`/`amd_iommu=off` is the intended recovery posture (arguably correct — minimal fallback — but with AMD-Vi off only in the main entry, the fallback boots with the *kernel default* IOMMU state; confirm that asymmetry is harmless on this firmware).

### B2. `/boot/loader/loader.conf`
```
default @saved
timeout 0
console-mode keep
editor no
```
- `default @saved` + `timeout 0` + `editor no`: confirm `@saved` resolves on systemd-boot with timeout 0; does a failed boot still let the user reach the menu, or does this create a recovery dead-end requiring external media? Flag the recovery-ergonomics trade (README points failures to live-USB → arch-chroot).

### B3. `/etc/nftables.conf` (exact ruleset — validate rule-by-rule)
```
#!/usr/bin/nft -f
flush ruleset
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        ct state established,related accept
        iif "lo" accept
        ct state invalid drop
        ip6 nexthdr ipv6-icmp icmpv6 type { nd-neighbor-solicit, nd-neighbor-advert, nd-router-advert, nd-router-solicit, echo-request, packet-too-big, time-exceeded, parameter-problem } accept
        icmp type { echo-reply, destination-unreachable, time-exceeded, parameter-problem } accept
        # [IFF RY_REMOTE_PLAY_PORTS=true:]
        tcp dport { 47984, 47989, 48010, 27036 } accept
        udp dport { 47998-48010, 27031-27036 } accept
    }
    chain forward { type filter hook forward priority filter; policy drop; }
    chain output { type filter hook output priority filter; policy accept; }
}
```
- **Rule-order:** `ct state invalid drop` sits AFTER `established,related accept` and `lo accept`. Confirm this cannot drop a valid loopback or established packet.
- **ICMPv6 set (8 types):** confirm minimal-but-complete for NDP + PMTUD + diagnostics. `mld-listener-query` / `nd-redirect` are absent — confirm intended on a LAN with a single router (no MLD-snooping dependency).
- **IPv4 ICMP (4 types):** echo-request (inbound ping) deliberately absent. Confirm `destination-unreachable` accept preserves IPv4 PMTUD (frag-needed). Confirm the asymmetric posture (IPv4 unpingable, IPv6 pingable) is intended.
- **`flush ruleset`:** confirm wiping the entire ruleset (not just inet filter) is safe vs docker/libvirt/podman nat tables on this host.
- **No rate-limiting on ICMP/new-conn:** confirm acceptable on a trusted LAN.
- Static `_vss_nft` (and runtime `_vrsv_nft_assert_ndp`) hard-fail if `nd-neighbor-solicit` is absent — treat that single rule as the IPv6 break-glass gate.

### B4. udev `99-ry-perf.rules` (exact — 3 rules)
```
ACTION=="add|change", KERNEL=="nvme[0-9]*n[0-9]*", ENV{DEVTYPE}=="disk", ATTR{queue/scheduler}="none"
ACTION=="add|change", SUBSYSTEM=="cpu", DEVPATH=="*/cpufreq", ATTR{cpufreq/energy_performance_preference}="balance_performance"
ACTION=="add", KERNEL=="card[0-9]", SUBSYSTEM=="drm", DRIVERS=="amdgpu", ATTR{device/power_dpm_force_performance_level}="auto"
```
- **NVMe + EPP `add|change`** (re-assert on change uevents) vs **GPU `add` only** (one-shot). Confirm the asymmetry is intended: EPP re-asserts after AC/DC; DPM=auto is set once (harmless one-shot). If a reviewer recommends `high`, note the `add`-only rule would NOT re-assert after a GPU reset — a reason `auto` is more robust.
- `KERNEL=="card[0-9]"` matches card0–card9 only. Confirm no multi-GPU/render-node (`renderD*`) concern on this single-iGPU host.
- Filename `99-` sorts after vendor `60-ioschedulers.rules` — confirm 99 wins (last-matching ATTR assignment).

### B5. `/etc/systemd/resolved.conf.d/99-cachyos-resolved.conf`
```
[Resolve]
MulticastDNS=no
LLMNR=no
DNSOverTLS=no
DNSSEC=allow-downgrade
```
- `99-` + basename `cachyos-resolved`: same filename = replace, not merge. If CachyOS ships its OWN `99-cachyos-resolved.conf` (DoH default), this file *replaces* it — confirm that is the intended override and not an accidental basename clash a CachyOS update would re-overwrite.

### B6. NetworkManager `99-cachyos-nm.conf` + dispatcher `logging.conf`
```
[device]
wifi.backend=wpa_supplicant
[connection]
wifi.powersave=2
[logging]
level=WARN
```
dispatcher: `[Service]` `LogLevelMax=notice` (drops info-level nm-dispatcher journal spam).
- Same basename-override caution as B5 (`99-cachyos-nm.conf`).
- Confirm `LogLevelMax=notice` on the dispatcher drop-in is the correct journald-noise fix (the comment notes `StandardError=null` is ineffective because dispatcher logs via journald) — validate against current NM-dispatcher behavior.

### B7. `/etc/iw-regdomain` + persistence path
```
COUNTRY=US
```
- **Persistence mechanism (name it precisely):** confirm `cachyos-iw-set-regdomain` is the CachyOS unit/script that reads `/etc/iw-regdomain` and runs `iw reg set` at boot — and that it still exists in current CachyOS. If CachyOS dropped it, the file is inert and the profile must switch to `wireless-regdb`/`crda` or a systemd unit. **Single most version-fragile external dependency in the profile.**

### B8. `/etc/bluetooth/main.conf` + `/etc/default/cpupower-service.conf`
```
[General]
FastConnectable=true
[Policy]
AutoEnable=true
ReconnectAttempts=3
```
cpupower-service.conf: `GOVERNOR='powersave'` (sourced by the cpupower service).
- Confirm `cpupower.service` on CachyOS actually sources `/etc/default/cpupower-service.conf` for `GOVERNOR` (path + var name). If CachyOS uses a different cpupower config path/unit, this file is inert and the governor falls to kernel default (the udev EPP rule still applies). `_vrsv_chk_cpupower_governor` asserts the running governor — validate the path.

### B9. `~/.config/MangoHud/MangoHud.conf` (exact, 17 active + 1 commented, ordered)
```
horizontal
legacy_layout=0
position=top-left
toggle_hud=Shift_R+F12
fps
frametime
frame_timing
gpu_stats
gpu_core_clock
gpu_temp
gpu_power
cpu_stats
# cpu_temp
cpu_mhz
vram
ram
font_size=20
text_outline
background_alpha=0.4
```
- `# cpu_temp` ships commented (v7.76.1) pending per-host hwmon resolution; re-enable is **`cpu_temp_sensor=k10temp`** (Round 2: corrected from `cpu_custom_temp_sensor`; MangoHud #1825 — note the open #1794 bug where `cpu_temp` zeroes `cpu_power` on Zen 5). Confirm `frame_timing` (graph) vs `frametime` (numeric) are both valid current keys and not redundant; confirm `toggle_hud=Shift_R+F12` syntax is current.

## C. Full verify subsystem (`--verify`) — 12 orchestrators

The top VERIFY block is the user-facing command set; the script's actual `--verify` runs 12 orchestrator functions across six sub-families. Recommendations that change a value MUST state which sub asserts it (and whether hard-fail or warn). The architecture (v7.77.1):

**Static (on-disk content):**
- `_verify_static_boot` — loader.conf, cmdline, sdboot.conf, mkinitcpio.conf strings (incl. `LINUX_FALLBACK_OPTIONS="quiet"`).
- `_verify_static_system` → subs `_vss_ntsync_modules` · `_vss_logind` · `_vss_nmdispatch` · `_vss_nm` · `_vss_sysctl` (key=value) · `_vss_regdom` · `_vss_bluetooth` · `_vss_udev` (all 3 rules, GPU_DPM_LEVEL-aware) · `_vss_nft` (**hard-fail on missing nd-neighbor-solicit**) · `_vss_modprobe` (mt7925e disable_aspm=1) · `_vss_known_benign`.
- `_verify_static_user` — ENV_VARS env.d + baloo + MangoHud (`_grep_mangohud_entry` accepts bareword OR key=value OR `#`-comment; asserts ≥1 directive + `fps`).
- `_verify_static_packages` · `_verify_static_services` · `_verify_static_syntax` · `_verify_static_checksum`.

**Known-benign (INFO-only, never fails) — `_kb_*` (5):** `_kb_modemmanager_masked` · `_kb_acp70_no_machine_driver` · `_kb_thunderbolt_nhi_unknown` · `_kb_no_battery_backlight` · `_kb_usb_mic_volume_curve`. Aggregated under `_vss_known_benign`.

**Runtime-kernel — `_verify_runtime_kparams`:**
- `_vrk_cmdline` — **generic loop asserting EVERY `KERNEL_PARAMS` token + `rw` is present in `/proc/cmdline`** (so `amd_iommu=off` is auto-asserted); + preemption-model INFO from dmesg `Dynamic Preempt:`.
- `_vrk_gpu_state` — `power_dpm_force_performance_level` == `$GPU_DPM_LEVEL` (auto) across `card*`.
- `_vrk_cpu_state` — scaling_driver=amd-pstate-epp, scaling_governor=$CPUPOWER_GOVERNOR, EPP=balance_performance, amd_pstate status/prefcore, boost=1.
- `_vrk_module_state` → subs `_vrkm_amdgpu` (hex-aware, no-ops without amdgpu.*), **`_vrkm_iommu`** (derives off/on from KERNEL_PARAMS; with `amd_iommu=off` hard-fails if iommu_groups present; dmesg AMD-Vi/DMAR correlation), `_vrkm_blacklist` (module_blacklist= scan); + usbcore.autosuspend, nvme_core ps_max_latency, zswap.enabled, nmi_watchdog, NVMe `[none]`.
- `_vrk_clocksource` — clocksource=tsc; HPET → **fail** + dmesg TSC-demotion (`Marking TSC unstable`) correlation.

**Runtime-services — `_verify_runtime_services`:** `_vrsv_chk_active_enabled` · `_vrsv_nft_assert_ndp` (NDP live) · `_vrsv_chk_nftables` · `_vrsv_chk_resolved` · `_vrsv_chk_cpupower_governor` · `_vrsv_sys_units` · `_vrsv_wifi_nm_backend` · `_vrsv_wifi_iwd_proc` · `_vrsv_wifi` · `_vrsv_masked_inactive`.

**Runtime-env — `_verify_runtime_env`:** `_vre_envvars` (`systemctl --user show-environment`) · `_vre_sysctl_runtime` (`/proc/sys`) · `_vre_tcp` (tcp_bbr) · `_vre_zram` (zram service + active swap; PASS/WARN — profile does not manage zram but asserts it) · `_vre_fstab` (ext4 has `noatime,lazytime,commit=10`) · `_vre_ntsync` (state dispatch) · `_vre_regdom` (`iw reg get`).

**Runtime-session — `_verify_runtime_session`:** `_vrs_nm_perms` (**system-connections 0600 root:root**) · `_vrs_vfat_skip` (skips perm check on vfat/undetermined boot) · `_vrs_installed_file_perms` (system 0644 / user 0600) · `_vrs_parent_dirs` · `_vrs_vulkan` (vulkan-radeon + lib32-vulkan-radeon).

Actionable for §C:
- Confirm `_vrkm_iommu`'s 0-groups expectation under `amd_iommu=off` is correct and its dmesg patterns (`AMD-Vi: (Enabled|Found|Interrupt)`, `DMAR: IOMMU enabled`, `Adding to iommu group`) are current; confirm it correctly stays silent when KERNEL_PARAMS carries no iommu directive.
- Confirm `_vre_zram` PASS/WARN policy (masked-zram → WARN, no-swap → WARN) — reconcile the manages-nothing-but-asserts-something tension.
- Confirm `_vrs_vfat_skip` carve-out (perms unverifiable on a vfat `/boot`) is not hiding a real perms regression on `/boot/loader/loader.conf`.
- Confirm `_vrk_clocksource` HPET-fail + TSC-demotion correlation is the right severity (fail vs warn) given `tsc=reliable` is on the cmdline.

## D. fstab rewrite (`_install_fstab_opts`) — normalization, not just append

The rewrite does more than add tokens:
- Adds `noatime,lazytime,commit=10` to ext4 entries (field 4 only); every other column and every non-ext4 row byte-preserved.
- **Strips conflicting tokens:** redundant `defaults`, `relatime`, `atime`, `strictatime`, and an existing `commit=` (rewritten to `commit=10`, not duplicated).
- Gates: line-count parity + size floor + mandatory `findmnt --verify`.
- **Refused (not corrected):** a symlinked or whitespace-split (malformed) `/etc/fstab`.

Confirm: (a) the rewrite touches ONLY ext4 (not the vfat ESP, not btrfs/xfs); (b) idempotent (re-run is a no-op on already-correct fstab); (c) atomic (tmp+rename) so a crash mid-rewrite cannot truncate this boot-critical file (it also takes a `.ry.bak`); (d) `commit=10` durability is acceptable given `fsck.mode=force` runs every boot (§3) — do the two interact (forced fsck offsets the wider commit window)?

## E. Preflight gate ordering (`_init_runtime` / `_install_preflight` / `_ir_*`)

Order matters for exit-code semantics:
- `_ir_resolve_root_uuid` → `EXIT_GEN_NOUUID 12` (sentinel) if cmdline render finds no UUID.
- Hardware gate (CPU match, override `RY_INSTALL_SKIP_HARDWARE_CHECK=1`, fail-closed on unreadable model).
- `_ir_validate_kernel_floor` (EXIT_PREFLIGHT 3) — kernel ≥ 6.18 (override `RY_INSTALL_SKIP_KERNEL_FLOOR_CHECK=1`, fail-closed on unreadable `uname -r`).
- `_ir_validate_counts` (3) — all 19 array counts.
- `_ir_validate_keys` (3) — scalar domains (true|false: BT_AUTO_ENABLE/BT_FAST_CONNECTABLE/RY_REMOTE_PLAY_PORTS; yes|no: SDBOOT_*/RESOLVED_*; int: LOADER_TIMEOUT/NM_WIFI_POWERSAVE/BT_RECONNECT_ATTEMPTS; ISO-3166 COUNTRY; non-empty: LOADER_*/SDBOOT_DEFAULT_ENTRY/RESOLVED_DNSSEC/NM_*/CPUPOWER_GOVERNOR/GPU_DPM_LEVEL/MKINITCPIO_COMPRESSION).
- `_ir_validate_post_hooks` (3) — every `_RY_POST_HOOKS` tag has a handler.
- Generator sentinels: `EXIT_GEN_NOFN 11`, `EXIT_GEN_NOUUID 12`, `EXIT_GEN_SYSCTL 13` (sysctl count mismatch in `_content_*sysctl`); `250/251/255` also never reach a process exit (surface as footer `gen_fail`).
- Advisories (non-fatal): mesa < 26.0 soft-floor; linux-firmware `20251125*` hard-warn (bad GC 11.5.1 MES 0x83) / `< 20260110` soft-warn (the revert-to-0x80 release); `_ry_check_umip_disabled`.

Confirm: (a) counts/keys/floor run BEFORE any disk write; (b) the two skip-override env vars are the only documented bypasses and each is scoped (kernel-floor and hardware only — counts/keys cannot be bypassed); (c) `PACTREE_TIMEOUT_S`, `BOOT_SPACE_CRIT`/`WARN` (200/500 MB), `ROOT_AVAIL_CRIT`/`WARN` (2/5 GB) are sane gates — are the boot-partition thresholds right for a systemd-boot ESP holding multiple kernels + fallback?

## F. Deeper-pass investigation deltas (new/updated actionable items)

1. **IOMMU live-effect assert (`_vrkm_iommu`, NEW):** confirm the 0-iommu_groups expectation under `amd_iommu=off`, the off/on derivation from KERNEL_PARAMS, and the dmesg correlation patterns. Highest-priority new verify surface.
2. **`amd_iommu=off` ROCm interaction — RESOLVED (Round 2):** disabling AMD-Vi does NOT break ROCm/HIP large-allocation or SVM on gfx1151 — `rocminfo` reports `IOMMU Support: None` on this iGPU and the Strix Halo LLM-toolbox community *recommends* `amd_iommu=off` for the 128 GB unified pool (lower latency, no SVM penalty). Keep as an accepted gaming/compute trade-off; do NOT escalate to a FIX or a compute-vs-gaming opt-in. The verify assert (`_vrkm_iommu`, item 1) remains the live check.
3. **linux-firmware floor anchors — RESOLVED (Round 2):** `20251125*` = bad GC 11.5.1 MES 0x83 (commit `9643cbf2…`); `20260110` = the revert to MES 0x80 (`c092c7487e…`), NOT a new fix. Known-good: 20251111 or 20260110+. Anchors are correct as written; only the rationale (revert vs fix) needed correction. Still-open: the exact release tag re-landing the corrected blob (`a0f0e52138…`).
4. **ntsync assert-only correctness:** confirm the CachyOS kernel builds `CONFIG_NTSYNC=y` so `/dev/ntsync` exists without the dropped autoload conf; else assert-only is a gap.
5. **MangoHud `cpu_temp` re-enable — RESOLVED (Round 2):** correct key is `cpu_temp_sensor=k10temp` (not `cpu_custom_temp_sensor`; MangoHud #1825); `k10temp`/Tctl is the Zen 5 sensor. Open caveat: enabling `cpu_temp` zeroes `cpu_power` on Zen 5 (#1794).
6. **`cachyos-iw-set-regdomain` existence (B7):** verify this external CachyOS unit still exists and reads `/etc/iw-regdomain`. Highest version-fragility in the profile.
7. **`cpupower.service` config path (B8):** verify CachyOS sources `/etc/default/cpupower-service.conf` `GOVERNOR`; `_vrsv_chk_cpupower_governor` asserts the running governor.
8. **`99-cachyos-*` basename overrides (B5/B6):** confirm same-filename replace (not merge) is intended and not a clash a CachyOS update would re-overwrite.
9. **`flush ruleset` blast radius (B3):** confirm wiping all nft tables is safe vs docker/libvirt/podman.
10. **`_vre_zram` / `_vre_fstab` coverage:** both in the VERIFY block; reconcile "zram out-of-scope" with "zram asserted in --verify".
11. **`LINUX_FALLBACK_OPTIONS="quiet"` recovery posture (B1):** confirm a param-stripped fallback (no `amd_iommu=off`, no `fsck.*`) is the intended recovery boot and `timeout 0`+`editor no` (B2) is not a dead-end.
12. **`card[0-9]` single-digit match (B4):** confirm no >9-card/renderD edge on this single-iGPU host.
13. **RTC write-back (`_ry_rtc_writeback`):** confirm `hwclock --systohc --utc` is safe and the `RTCInLocalTZ=yes` defer-to-systemd guard is correct.


---

# Deepest-pass appendix (§G–§L) — robustness & correctness audit surface

§1–§13 audit *what the profile configures (and what it omits)*; §A–§F audit *what the script writes and asserts*. This final layer audits *whether the installer is safe to run at all* — the atomic-write, locking, privilege, rollback, and signal machinery. These are correctness questions, not tuning. A reviewer must confirm each guarantee holds on current fish (3.6 floor) / CachyOS, and flag any TOCTOU, fail-open, or partial-write window. Every mechanism below is quoted from `ry-install.fish` v7.77.1.

## G. Atomic-write guarantees (`_awf_*`)

Write path per managed file: `_awf_render_to_tmp` → `_awf_symlink_check` → `_awf_finalize_mv`; boot/backup targets add `_awf_make_backup` (pre) + `_awf_postwrite_verify_restore` (post).

- **tee-to-tmp with `$pipestatus`:** content generator piped into `_as $use_sudo tee -- "$tmpfile"`; `$pipestatus[1]` (generator) and `$pipestatus[2]` (tee) checked separately, mapping generator failures to `EXIT_GEN_NOFN/NOUUID/SYSCTL`. Confirm fish `$pipestatus` still distinguishes the two stages and a generator returning non-zero never leaves a partial tmpfile promoted.
- **Post-write symlink-swap probe (`_awf_symlink_check`):** after writing tmp, re-tests whether tmp was replaced by a symlink (rc 0/1/2 = symlink/not/sudo-lapse), aborting on swap. Confirm the probe closes the window (any gap between the symlink check and the `mv -T`?).
- **`mv -T` atomic rename (`_awf_finalize_mv`):** chmod tmp → re-assert `sudo -n true` (credential-lapse guard) → `mv -T -- tmp dst`. Confirm `mv -T` is atomic on the same filesystem (tmp created in dst's parent so rename is same-FS — verify this holds for `/boot` on vfat, where rename atomicity differs).
- **Post-write byte re-read + restore (`_awf_postwrite_verify_restore`):** re-runs the generator, reads installed bytes (`_installed_bytes`, tri-state rc), compares; on mismatch restores `.ry.bak` via `mv -T`. Confirm: (a) generator determinism (the sysctl generator side-effects `_RY_SYSCTL_BAD_ENTRIES`; preflight already refuses `_RY_BACKUP_TARGETS` members using side-effecting generators — confirm that guard is complete); (b) restore `mv -T` is itself atomic; (c) `string collect --no-trim-newlines --allow-empty` preserves trailing-newline-sensitive comparisons.
- **Backup only for `_RY_BACKUP_TARGETS` (2):** loader.conf + mkinitcpio.conf. Every other managed file relies solely on atomic-write (no .bak). Confirm a post-write mismatch on a NON-backup file (e.g. nftables.conf) is detected but cannot be auto-restored — is that the right risk posture for the firewall ruleset?

Actionable: confirm the symlink-check→mv window is non-exploitable; confirm `/boot` vfat rename atomicity; confirm generator determinism for all post-write-verified files.

## H. Instance lock & PID-recycle TOCTOU (`_acquire_lock*`, `_lock_pid_started_after`)

Lock = atomic `mkdir "$LOCK_DIR"` (umask 0077, chmod 700) + pidfile via `mktemp`+`mv -Tf` (chmod 600). Stale reclaim bounded (3 attempts) and fail-closed.

- **Atomic mkdir as the lock primitive:** confirm race-free vs a competing instance on the same host.
- **PID-recycle detection (`_lock_pid_started_after`):** reads `/proc/PID/stat` field 22 (starttime ticks) + `/proc/stat` btime ÷ USER_HZ (`getconf CLK_TCK`, with **CONFIG_HZ recovery from `/proc/config.gz`** if getconf absent), reclaims ONLY if holder start > pidfile mtime + 2s. **Fail-closed**: any unparseable field → treat as live, refuse reclaim. Confirm: (a) the `string replace -r '^.*\) '` correctly handles a `comm` containing `) ` (the classic `/proc/stat` parsing trap); (b) field-22 indexing survives a comm with spaces/parens; (c) the +2s slack is sufficient vs clock granularity but tight enough to catch a recycled PID.
- **Re-read-before-rm TOCTOU guard:** before `rm -rf` of a stale lock, pidfile is re-read and compared to the decision-time value; changed → abort. Confirm this closes the reclaim race.
- **Symlink refusal:** symlinked `$LOCK_DIR` → reclaim refused; `rm -rf --preserve-root`. Confirm no path-traversal.
- **kill -0 EPERM → /proc branch:** unsignalable-but-alive PIDs (different UID) detected via `/proc/PID` and NOT reclaimed. Confirm correct.

Actionable: confirm `/proc/stat` comm-parsing robustness; confirm fail-closed on every USER_HZ/btime/starttime read failure; confirm the re-read-before-rm window is closed on current kernels.

## I. Privilege handling (`_as`, `_run`, `_is_symlink`, `_installed_bytes`)

- **`_as use_sudo` / `sudo -n` everywhere:** all privileged ops use non-interactive `sudo -n`; credential lapse re-checked immediately before each critical write (`mv`, mkinitcpio revert). Confirm NO interactive sudo prompt mid-run (would hang an unattended install) and that mid-run credential expiry fails safe (aborts the file, no partial-write).
- **Tri-state rc 0/1/2 (drift vs sudo-lapse):** `_is_symlink`, `_installed_bytes`, `_ry_content_bytes` return 2 specifically for sudo-cache-lapse so callers distinguish "file differs" from "couldn't read due to expired sudo." Confirm every caller branches on rc 2 (a 2→1 collapse would misreport a sudo lapse as drift). Spot-check across callers.
- **`_run` timeout enforcement:** `_run` wraps commands with logging + capture + timeout (`RY_RUN_TIMEOUT`, default 3600 s, 0 disables). Confirm pacman/mkinitcpio/sdboot-manage/paccache/updatedb/pkgfile are EXEMPT from the cap (README states they are) so a slow mirror/large initramfs is not killed; `PACTREE_TIMEOUT_S` governs pactree only.

Actionable: audit every rc-2 caller for the drift-vs-lapse distinction; confirm no interactive-sudo hang path; confirm the long-op timeout exemptions are complete.

## J. Boot-wipe gate & boot-critical rollback (`_irb_taint_gate`, `_install_rebuild_boot`)

The single most dangerous operation: `SDBOOT_REMOVE_EXISTING=yes` makes `sdboot-manage gen` wipe all `loader/entries/` (foreign BLS included). Gate sequencing:
1. `_irb_taint_gate` → `_check_boot_taint_gate`: if `_RY_BOOT_TAINTED=true` OR the mkinitcpio.conf revert failed, **SKIP `mkinitcpio -P` entirely** and return `EXIT_BOOT_CRIT` (4). Confirm the taint flag is set on any prior boot-critical write failure so a half-written cmdline/mkinitcpio never reaches `mkinitcpio -P`.
2. `mkinitcpio -P` failure → abort remaining steps, `EXIT_BOOT_CRIT`, skip post-mki.
3. **`$BOOT` resolution refusal:** if `SDBOOT_REMOVE_EXISTING=yes` AND `_resolve_boot_path` returns empty (bootctl + findmnt both failed AND `/boot` missing), **refuse the boot-wipe gate** — `EXIT_BOOT_CRIT` rather than run `sdboot-manage` against an unresolved `$BOOT`. A non-vfat `/boot` ESP also refuses sdboot (exit 4). This guard prevents wiping the wrong/no target.
4. Post-rebuild sanity (`_preflight_boot_sanity`): vmlinuz + initramfs + entries must exist or `EXIT_BOOT_CRIT`.

Confirm: (a) NO path where `sdboot-manage REMOVE_EXISTING=yes` runs against an unverified `$BOOT`; (b) a mkinitcpio failure cannot leave a new cmdline with a stale/absent initramfs — trace the Phase-3 cmdline-write vs Phase-5 initramfs-rebuild ordering window and confirm `LINUX_FALLBACK_OPTIONS="quiet"` (§B1) covers a boot with new cmdline + old initramfs (note: the fallback entry carries neither `amd_iommu=off` nor `fsck.*`, so a fallback boot reverts to kernel-default IOMMU — confirm benign); (c) `EXIT_BOOT_CRIT` aborts ALL remaining phases (no Finalize after a boot failure).

**mkinitcpio rollback (`_ip_snapshot_mkinitcpio` / `_mkinitcpio_revert`):** before `pacman -Syu`, mkinitcpio.conf is snapshotted to `/run/ry-install` (0700 root:root, mktemp). On pacman failure, `_mkinitcpio_revert` restores via same-`/etc`-FS mktemp + `_mr_copy_cmp_verify` (cp + byte-exact `cmp`) + `_mr_chmod_chown_mv` (atomic mv, `--reference=destination` perms). Confirm the snapshot→revert path is complete (snapshot in `/run` tmpfs lost on reboot — acceptable since revert is same-boot; revert tmp in `/etc` same-FS atomic; symlink-checked). Flag if a pacman partial-transaction could desync mkinitcpio.conf from installed modules without triggering revert.

Actionable: trace the cmdline(Phase 3)-vs-initramfs(Phase 5) ordering window; confirm the `$BOOT`-unresolved + non-vfat refusal is the only path to `sdboot-manage REMOVE_EXISTING`; confirm `EXIT_BOOT_CRIT` is terminal.

## K. Signal & exit teardown (`_cleanup`, `_teardown`, `_do_cleanup`)

- **Signal handlers:** `_cleanup` traps INT/TERM/HUP/QUIT/ABRT with correct **128+N exit codes** (130/143/129/131/134), idempotent via `_CLEANUP_DONE`. SIGPIPE (`_cleanup_pipe`) marks output broken and continues JSONL-only. `fish_exit` (`_cleanup_on_exit`) ensures teardown on normal exit, preferring `_INTENDED_EXIT_CODE` → `_RY_INSTALL_LAST_EXIT` → `$status`.
- **Cleanup order (`_do_cleanup`):** kill children → mkinitcpio revert → tmpfile sweep → filesystem sweep → **lock release (sweeps run while lock still held)** → erase globals. Confirm: (a) children reaped BEFORE revert so revert never races a live pacman; (b) lock released LAST so no second instance starts mid-sweep; (c) the `_RY_TMPDIR_GLOBS` (6 patterns) match every tmpfile created (ry-sudo-err, ry-tee-err, ry-run, ry-argparse-err, ry-fstab-tee-err, ry-fstab-awk-err) — a missed glob leaks; an over-broad glob deletes another instance's tmp. Verify the glob set is exactly the created set.

Actionable: confirm SIGKILL (uncatchable) is the only signal bypassing cleanup and the lock's stale-reclaim (§H) recovers from a SIGKILLed holder; confirm the tmpdir glob set exactly matches the created-tmpfile set.

## L. pacman transaction safety (`_ip_pacman_invoke`)

- **Full `-Syu --needed` only (no partial upgrades):** first pass `-Syu`, retry `-Syyu` (forced db re-sync for mirror staleness), `--noconfirm`. Confirm the retry only addresses transient staleness and never masks a real conflict (the second failure must be fatal and surfaced).
- **db.lck pre-check:** refuses if `/var/lib/pacman/db.lck` exists; never removes it itself (instructs the user). Confirm checked before any package op.
- **PKGS_DEL via `-Rns` (rdep-aware):** an external dependant skips + logs rather than cascading. Confirm `-Rns` never triggers a partial-upgrade state and that full `-Syu` is mandatory (script never issues `-S <pkg>` without `-yu`).

Actionable: confirm retry-then-fatal semantics; confirm no self-removal of db.lck; confirm PKGS_DEL `-Rns` cannot induce a partial-upgrade.

## Robustness verdict (required, separate from §1–§13)

Add a final **ROBUSTNESS** verdict block:
- For each of §G–§L: PASS (guarantee holds) / GAP (window or fail-open found) / UNCERTAIN (cannot confirm against current fish/kernel without testing).
- Any GAP in §G/§H/§J (atomic-write, lock, boot-wipe) is release-blocking and outranks every tuning finding — surface it first regardless of IMPACT/RISK on config items.
- This layer is correctness, not preference: there is no "deliberate trade-off" defense for a partial-write window or fail-open lock. FIX applies normally here (flag-don't-FIX is for config values, not safety invariants).

Sources for §G–§L: man7.org (mkdir(2), rename(2)/mv -T atomicity, proc(5) stat fields, sysconf CLK_TCK), fishshell.com/docs (`$pipestatus`, `string collect`, `--on-signal`, `--on-event fish_exit`), wiki.archlinux.org (pacman partial-upgrade policy, mkinitcpio, systemd-boot), docs.kernel.org (/proc/stat btime, USER_HZ).
