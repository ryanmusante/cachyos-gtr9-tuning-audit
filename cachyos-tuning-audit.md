# Deep-Research Prompt — ry-install Tuning (CachyOS · Beelink GTR9 Pro)

Target: `ry-install.fish` **v7.75.1** (attached). Source of truth: script > README > CHANGELOG.

**Platform:** Beelink GTR9 Pro · Ryzen AI Max+ 395 "Strix Halo" (Zen 5, 16C/32T, gfx1151) · Radeon 8060S (40 RDNA 3.5 CUs) · XDNA 2 NPU · 128 GB LPDDR5X-8000 unified (≤96 GB as VRAM) · dual M.2 NVMe (ext4) · dual 10 GbE (RTL8127) + Wi-Fi 7 (MT7925) + BT 5.4 · 140 W TDP · CachyOS · systemd-boot.

**Counts (from `_ir_validate_counts`):** KERNEL_PARAMS 16 · ENV_VARS 11 · SYSCTL_VALUES 9 · PKGS_ADD 17 · PKGS_DEL 9 · MASK 10 · EXPECTED_SERVICES 5 · managed files 18 (15 system + 3 user) · MangoHud 18 directives · MKINITCPIO_HOOKS 11 · LOGIND_IGNORE_KEYS 8 · NTSYNC_MODLOAD_CONFS 3.

**Hard floors:** KERNEL_MIN **6.18** (preflight hard-fail; RTL8127 suspend-hang fix `ae1737e7339b` + r8169 support land ≥6.18) · CPU gate `Ryzen AI Max` · soft Mesa < 26.0 warn.

## What changed since v7.70.1 (re-audit focus)

The prior prompt targeted v7.70.1. The following are NEW or INVERTED and need fresh evidence — do not carry forward the old conclusions on these:

1. **KERNEL_MIN 6.18 is now a HARD preflight floor** (was "no hard floor"). Re-establish whether 6.18 is the correct concrete floor for the RTL8127 suspend-hang fix `ae1737e7339b` + r8169 landing, or whether a different version is the true gate.
2. **GPU_DPM_LEVEL flipped `high` → `auto`** (parameterized, v7.71.0). The entire §6 GPU clock-floor premise is inverted: the profile NO LONGER pins SCLK. Re-evaluate `auto` vs `high` on a shared 140 W package given EPP=balance_performance.
3. **ntsync is now MANAGED** (was "observed, not deployed"): 3 modules-load.d autoload confs + a builtin|loaded|loaded_nodev|missing state machine. Confirm the autoload posture is correct and the device-missing path is handled.
4. **RY_REMOTE_PLAY_PORTS gate added** (default `false`): when true, appends Sunshine/Moonlight + Steam Remote Play ports to the nftables input chain. Validate the exact port set.
5. **PROTON_FSR4_RDNA3_UPGRADE=1 re-added** (was deliberately removed in v7.70.1). Confirm current Proton-CachyOS FSR4-on-RDNA3.5 support — does this env var do anything today, or is it inert/harmful?
6. **4 kernel params added:** `btusb.enable_autosuspend=n`, `processor.max_cstate=1`, `fsck.mode=force`, `fsck.repair=yes`. All four need first-time validation.
7. **2 sysctl keys dropped** as vendor duplicates: `vm.page-cluster=0`, `vm.vfs_cache_pressure=50`. Confirm 70-cachyos-settings.conf actually sets these to the same values (else removal is a silent behavior change).
8. **MangoHud restored 3 directives** (`gpu_power`, `text_outline`, `toggle_hud=Shift_R+F12`); `cpu_temp` and `fps_metrics` remain absent. 18 directives total.
9. **New modprobe drop-in `60-ry-mt7925e.conf`** (`options mt7925e disable_aspm=1`) — symptomatic MT7925 coredump/BT-reconnect/assoc-fail mitigation. Confirm ASPM-disable is the current correct workaround and whether an upstream fix has since landed.
10. **CPUPOWER_GOVERNOR and EPP unchanged** (powersave + balance_performance) — but v7.75.1 reasserts them as a pairing. Treat under the governor/EPP special case (do not flag powersave as a regression).

## Mission

Evaluate every config decision against current upstream sources for this exact silicon. Return a prioritized, evidence-backed tuning report. The profile deliberately trades power-saving and host firewalling for performance and latency; confirm each choice is current and correct, surface anything superseded or harmful, and quantify safety deltas without second-guessing intentional design.

## Rules

1. Item-by-item, hardware-anchored to gfx1151 / Zen 5 / RDNA 3.5 / CachyOS / 128 GB unified / dual 10 GbE.
2. Respect deliberate trade-offs: **flag and quantify, do not auto-FIX.** Reserve FIX for incorrect, superseded, deprecated, or harmful values.
3. Rate IMPACT × RISK (High/Med/Low). Default KEEP when impact is marginal and risk is non-trivial.
4. Never invent params, flags, keys, options, or URLs. Cite a source or mark UNCERTAIN.
5. Flag every source conflict and state which is trusted.
6. Give exact versions (kernel / Mesa / linux-firmware / pkg) and exact before→after, mapped to the in-script global.

## Output

- **Findings matrix** (box-drawn Unicode, code fence, grouped by section): ITEM · CURRENT (v7.75.1 value) · CALL (KEEP/TUNE/FIX/UNCERTAIN) · RECOMMENDED · IMPACT · RISK · EVIDENCE (URL + version/date/commit).
- **Before→after** for each TUNE/FIX: exact current string, exact replacement, in-script global.
- **VERIFY block** (post-reboot commands, below).
- **Security delta vs CachyOS defaults** (ordered, below).
- **Verdict:** one per section (OPTIMAL/TUNE/FIX) plus overall (PASS/PASS-WITH-FIXES/FAIL).
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
cat /sys/.../clocksource0/current_clocksource               # tsc
cat /sys/block/nvme0n1/queue/scheduler                      # [none] (adjust node)
cat /sys/class/drm/card*/device/power_dpm_force_performance_level   # auto (DPM floor un-pinned in v7.71.0)
cat /proc/cmdline | rg -o 'iommu=\S+'                       # iommu=pt
cat /proc/cmdline | rg -o 'clearcpuid=\S+'                  # clearcpuid=514 (UMIP off)
cat /proc/cmdline | rg -o 'processor.max_cstate=\S+'        # 1 (C-state cap)
cat /proc/cmdline | rg -o 'fsck\S+'                         # fsck.mode=force fsck.repair=yes
ls -l /dev/ntsync                                           # present (ntsync now managed: autoload confs shipped)
cat /usr/lib/modules-load.d/ntsync.conf 2>/dev/null         # ntsync (one of 3 candidate autoload confs)
cat /etc/modprobe.d/60-ry-mt7925e.conf                      # options mt7925e disable_aspm=1
vulkaninfo | rg -i 'driverName|deviceName'                 # RADV / Radeon 8060S; confirm uma heap
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc vm.max_map_count vm.compaction_proactiveness vm.swappiness
findmnt -no OPTIONS /                                       # noatime,lazytime,commit=10
iw reg get | rg -i country                                 # US
cat /etc/iw-regdomain                                       # COUNTRY=US
sudo nft list chain inet filter input                      # policy drop + ICMPv6 NDP/PMTUD + IPv4 diag-only (no inbound ping); +remote-play ports IFF RY_REMOTE_PLAY_PORTS=true
systemctl is-enabled bluetooth.service                     # enabled
printenv MANGOHUD                                          # 1
```

Hard `--verify` asserts (mismatch → exit 1): scaling_driver, scaling_governor=powersave, EPP=balance_performance, amd_pstate status/prefcore, boost, clocksource, regdom (`/etc/iw-regdomain`), nftables `nd-neighbor-solicit` present, mt7925e `disable_aspm=1` drop-in. NOTE: GPU `power_dpm_force_performance_level` is now `auto`, not `high` — the prior drift-on-≠high assert is gone; verify it reads `auto`. Warn-level only: ZRAM, live nftables NDP cross-check, ntsync runtime device state, iwd runtime state (opt-in). Advisory INFO (never fails): `_vss_known_benign` (masked ModemManager D-Bus noise, ACP70 no-machine-driver, boltd NHI-unknown, no battery/backlight sysfs, USB-mic volume curve). No boot-time, THP, KSM, `ttm.*`, drirc, or `radv_enable_unified_heap_on_apu` checks exist (removed) — do not verify them.

### Security delta (ordered)

1. **UMIP off** (`clearcpuid=514`) — descriptor-table base leak, kernel tainted; headline open reduction.
2. **split_lock_detect=off** — a misbehaving app can degrade the system.
3. **Plaintext DNS** (`DNSOverTLS=no`, `DNSSEC=allow-downgrade`) reverting the CachyOS DoH default — DNS observable and spoofable on-path.
4. **Optional inbound remote-play ports** (`RY_REMOTE_PLAY_PORTS`, default OFF) — when enabled, opens Sunshine/Steam stream ports; quantify the exposure of the opt-in.
5. **Firewall default-deny-inbound ships** (nftables; no inbound IPv4 ping) — net positive.
6. **IOMMU restored** (`iommu=pt`, AMD-Vi on) — the prior `amd_iommu=off` reduction is closed.

## Investigation (ordered by installer phase)

### 1. Platform baseline and version floors

Current: **hard kernel floor KERNEL_MIN 6.18** (preflight `_ir_validate_kernel_floor`, override `RY_INSTALL_SKIP_KERNEL_FLOOR_CHECK=1`); CPU gate `Ryzen AI Max` (override `RY_INSTALL_SKIP_HARDWARE_CHECK=1`); soft Mesa < 26.0 warn (preflight); MES 0x86 fix via linux-firmware + mkinitcpio-firmware.

- **Re-validate the 6.18 floor:** confirm `ae1737e7339b` (RTL8127 suspend/shutdown-hang fix) and r8169 RTL8127 support both land at/above 6.18 mainline, and map to the CachyOS kernel. Is 6.18 the true minimum, or is the fix backported lower? State the exact landing version and whether the floor should move.
- Confirm the soft Mesa 26.0 floor matches current RADV guidance; enumerate open gfx1151 RADV issues and state whether 26.0 is still the right threshold.
- Confirm gfx1151 reports `uma:1` natively, so the removed drirc override is genuinely redundant on current Mesa.
- Give the exact linux-firmware version first carrying the MES 0x86 blob; check the installed version.
- Sources: wiki.cachyos.org, docs.kernel.org gpu/amdgpu, gitlab.freedesktop.org/mesa (gfx1151), git.kernel.org linux-firmware + r8169.

### 2. Packages

PKGS_ADD (17): nvme-cli, cachyos-gaming-meta, cachyos-gaming-applications, lib32-mesa, mkinitcpio-firmware, fd, sd, dust, procs, bottom, htop, git-delta, lm_sensors, rtkit, realtime-privileges, ddcutil, nftables. PKGS_DEL (9): plymouth stack + micro + cachyos-micro-settings + cachy-update + kdeconnect. AUR: none. Vulkan (chwd): vulkan-radeon, lib32-vulkan-radeon. (Note: the v7.74.0 advisory x86-64-v4 repo-tier probe was removed — repo tier is no longer assessed by the script.)

- Confirm cachyos-gaming-meta and -applications supply RADV/Proton/gamescope/MangoHud/GameMode, and that the meta pulls MangoHud (the profile also ships MangoHud.conf — confirm no conflict).
- `rtkit`: confirm RealtimeKit is correct alongside realtime-privileges for PipeWire thread priority; does `rtkit-daemon.service` need enabling, or is it socket-activated?
- Confirm `lib32-mesa` is still needed alongside `lib32-vulkan-radeon`, or now redundant.
- Confirm plymouth, micro, and cachy-update removal has no dependency fallout.
- Since the repo-tier probe was removed: independently state whether znver/x86-64-v4 (AVX-512) repos benefit this build over v3, as a one-line advisory — but note the script no longer acts on it.
- Sources: wiki.cachyos.org, wiki.archlinux.org/Gaming + PipeWire + RealtimeKit.

### 3. Kernel cmdline (16)

`8250.nr_uarts=0 amd_pstate=active btusb.enable_autosuspend=n clearcpuid=514 fsck.mode=force fsck.repair=yes iommu=pt nowatchdog nvme_core.default_ps_max_latency_us=0 pcie_aspm.policy=performance processor.max_cstate=1 quiet split_lock_detect=off tsc=reliable usbcore.autosuspend=-1 zswap.enabled=0`

NEW since v7.70.1 — validate each for the first time:

- **processor.max_cstate=1:** capping C-states trades idle power for wake-latency/jitter. Confirm this is beneficial on Strix Halo for frametime consistency, quantify the idle-power and thermal cost on a 140 W package, and confirm it does not fight amd_pstate=active or starve boost headroom. Is `1` the right cap vs leaving C-states to the platform?
- **btusb.enable_autosuspend=n:** confirm disabling BT USB autosuspend is the correct fix for MT7925/BT 5.4 reconnect/stutter (alongside the mt7925e ASPM drop-in §8/§11). Does it overlap with `usbcore.autosuspend=-1`, making one redundant?
- **fsck.mode=force + fsck.repair=yes:** forcing fsck every boot on dual NVMe ext4 — confirm this is safe and intended (boot-time cost, behavior on a dirty/large filesystem, interaction with systemd fsck units and the `fsck` mkinitcpio hook). Auto-repair without prompt: confirm no data-loss risk on ext4. Is "force every boot" the right durability posture, or should it be periodic?

Carry-forward (confirm still current):

- **iommu=pt:** confirm AMD-Vi left enabled by default, so dropping explicit `amd_iommu=on` is genuinely redundant; quantify gaming/compute cost of `pt`; confirm recommended for ROCm + USB4 isolation.
- **clearcpuid=514 (UMIP off):** benefit (avoiding umip_printk stutter under anti-cheat/Wine) vs security cost (descriptor-table leak, taint). Confirm whether current Proton/EAC/BattlEye actually trip UMIP emulation on Zen 5. Recommend dropping if no stutter observed.
- **amd_pstate=active:** confirm recommended on Zen 5; interaction with powersave governor + balance_performance EPP (§6).
- **split_lock_detect=off:** perf vs stability; current default; blast radius.
- No `preempt=` pinned: confirm leaving preemption to CachyOS BORE/EEVDF costs nothing.
- Zero amdgpu/ttm module params: confirm hands-off GPU-param posture is correct.
- Validate the rest: tsc=reliable, nowatchdog, 8250.nr_uarts=0, usbcore.autosuspend=-1, nvme_core.default_ps_max_latency_us=0, pcie_aspm.policy=performance, zswap.enabled=0.
- Sources: docs.kernel.org kernel-parameters + pm/amd-pstate + x86 UMIP + admin-guide (processor.max_cstate, fsck), wiki.archlinux.org/AMDGPU + IOMMU + fsck.

### 4. Bootloader and initramfs

loader.conf: default @saved, timeout 0, console-mode keep, editor no. sdboot-manage: DEFAULT_ENTRY manual, OVERWRITE/REMOVE_EXISTING/REMOVE_OBSOLETE yes, LINUX_FALLBACK_OPTIONS "quiet". mkinitcpio: MODULES=(amdgpu), HOOKS (11) = base systemd autodetect microcode modconf kms keyboard sd-vconsole block filesystems fsck, COMPRESSION zstd.

- Verify HOOKS order with the systemd hook (microcode/kms/sd-vconsole/block placement); confirm amdgpu + kms is recommended early-KMS for this GPU. With `fsck.mode=force` now on the cmdline (§3), confirm the `fsck` hook + fsck.repair handshake produces no boot prompt or hang.
- Confirm zstd level for NVMe; quantify any higher-level boot-time win (no boot-time target pinned).
- timeout 0 + DEFAULT_ENTRY manual + REMOVE_EXISTING=yes wipes foreign BLS entries (EFI loaders untouched); confirm current and intended.
- Confirm sdboot-manage is current and maintained vs kernel-install/UKI (UKI out of scope).
- Sources: wiki.archlinux.org/Mkinitcpio + systemd-boot, sdboot-manage upstream.

### 5. GPU / Vulkan / gaming

No drirc shipped (gfx1151 reports uma:1 natively). No ttm/modprobe params (kernel auto-sizes GTT). ENV_VARS (11): AMD_VULKAN_ICD=RADV, DXVK_LOG_LEVEL=none, DXVK_LOG_PATH=none, MANGOHUD=1, MESA_SHADER_CACHE_MAX_SIZE=16G, PROTON_ENABLE_WAYLAND=1, **PROTON_FSR4_RDNA3_UPGRADE=1 (re-added v7.71.0)**, PROTON_LOCAL_SHADER_CACHE=1, VKD3D_DEBUG=none, VKD3D_SHADER_DEBUG=none, WINEDEBUG=-all.

- **PROTON_FSR4_RDNA3_UPGRADE=1 (re-added — the inverse of the v7.70.1 removal):** the prior prompt's non-goals forbade reinstating this. It is now back. Confirm current Proton / Proton-CachyOS actually consume this variable to upgrade FSR3.1→FSR4 on RDNA 3.5 (gfx1151), the minimum Proton-CachyOS version, and whether it is a no-op or harmful on titles without FSR. If unverified upstream, flag FIX-to-remove; if verified, KEEP and cite.
- **ntsync (NOW MANAGED — 3 autoload confs + state machine):** the prior prompt's "largest unmanaged gaming surface" is now managed. Confirm: (a) ntsync is the current Wine-sync mechanism vs esync/fsync; (b) which kernel mainlined it and which CachyOS kernel ships `CONFIG_NTSYNC=y` (builtin) vs module; (c) that shipping 3 candidate modules-load.d paths (`/usr/lib/modules-load.d/ntsync.conf`, `/usr/lib/modules-load.d/10-ntsync.conf`, `/etc/modules-load.d/ntsync.conf`) is sane and non-conflicting; (d) Proton consumes `/dev/ntsync` and improves frametimes on 16C/32T; (e) the `loaded_nodev` state (module loaded but `/dev/ntsync` absent) is a real failure mode worth the explicit handling. Is autoload even needed if the CachyOS kernel builds it in?
- **RADV unified heap (drirc removed):** confirm current RADV on gfx1151 reports uma:1 and treats the heap as unified without the override. If not, flag a regression.
- **GTT sizing (ttm removed):** confirm the installed kernel auto-sizes GTT sensibly (~62 GiB ceiling). Right for both large-VRAM gaming and ROCm/llama.cpp wanting ~96 GiB, or does removing the cap under-provision compute? If a workload needs >~62 GiB, determine the current correct mechanism (BIOS UMA carveout vs ttm.* vs sysfs) and whether an opt-in knob should return. Confirm amdgpu.gttsize still deprecated.
- PROTON_ENABLE_WAYLAND=1: maturity and fallback on current Proton.
- AMD_VULKAN_ICD=RADV: confirm it reliably forces RADV vs VK_DRIVER_FILES.
- MESA_SHADER_CACHE_MAX_SIZE=16G + PROTON_LOCAL_SHADER_CACHE=1: confirm no conflict, sane sizing.
- MANGOHUD=1 global enable: confirm low-overhead vs per-launch, clean with gamescope/GameMode.
- Sources: docs.mesa3d.org (RADV, APU heap), gitlab.freedesktop.org/mesa + drm/amd (GTT auto-sizing kernel version), github Proton/Proton-CachyOS (FSR4, ntsync), amd.com ROCm, wiki.cachyos.org.

### 6. CPU performance and power

amd_pstate=active; governor **powersave** (honors EPP under active mode); EPP **balance_performance** via udev (add|change, re-asserts after AC/DC); **GPU clock-floor GPU_DPM_LEVEL=auto (was `high` in v7.70.1; un-pinned in v7.71.0, add-only udev)**. Masked: power-profiles-daemon, ananicy-cpp, modemmanager. Installed: realtime-privileges, rtkit.

- **GPU_DPM_LEVEL=auto (INVERTED from v7.70.1's `high`):** the profile NO LONGER pins SCLK to high. Re-evaluate from scratch: (a) does leaving DPM at `auto` cost frametime/1%-lows on gfx1151 vs the old `high`, or was pinning high only burning idle power? (b) on a shared 140 W package with EPP=balance_performance, does `auto` correctly let the firmware arbitrate CPU↔GPU power, avoiding the boost-theft the old `high` risked on CPU-bound titles? (c) is there any title class where `high` still wins enough to warrant a documented opt-in? The drift assert that treated ≠high as failure is gone — confirm `auto` is the right default and quantify the delta vs `high`.
- **governor=powersave + EPP=balance_performance** is the EPP-honoring config under active mode (performance governor pins max and ignores EPP). Do not flag powersave as a power regression. Confirm the active + powersave + balance_performance triple is the documented EPP-honoring max-perf config on Zen 5; state whether balance_performance (mid-bias) leaves boost residency on the table vs EPP=performance, and whether mid-bias is the better thermal/latency trade on a 140 W mini-PC.
- Confirm udev add|change pinning is robust vs one-shot; prefcore=enabled + boost=1 correct on Strix Halo.
- Mask power-profiles-daemon: confirm no needed platform_profile path lost; does CachyOS expect ppd? Consider tuned.
- Mask ananicy-cpp: confirm net win for gaming.
- Mask modemmanager: confirm no cellular HW, masking loss-free (note: masked ModemManager now also produces a known-benign D-Bus log line flagged by `_vss_known_benign` §12 — confirm it is truly cosmetic).
- Sources: docs.kernel.org pm/amd-pstate (active + EPP + governor interaction), wiki.archlinux.org/CPU_frequency_scaling + AMDGPU (power_dpm_force_performance_level), freedesktop ppd.

### 7. Memory and storage

zswap.enabled=0; NVMe scheduler none (udev `99-ry-perf.rules`, sorts after vendor 60-ioschedulers.rules); **SYSCTL_VALUES (9):** net.core.default_qdisc=fq, net.core.netdev_budget=600, net.core.netdev_budget_usecs=5000, net.ipv4.tcp_congestion_control=bbr, net.ipv4.tcp_notsent_lowat=16384, net.ipv4.tcp_slow_start_after_idle=0, vm.compaction_proactiveness=0, vm.max_map_count=2147483642, vm.swappiness=150 (priority 95 `95-ry-overrides.conf` after vendor 70-cachyos-settings.conf); fstab ext4 noatime,lazytime,commit=10; THP/KSM/systemd-oomd left to CachyOS.

**DROPPED since v7.70.1 (v7.73.0, as vendor duplicates):** `vm.page-cluster=0` and `vm.vfs_cache_pressure=50` are no longer set by the profile.

- **Verify the drop is truly a no-op:** confirm CachyOS `70-cachyos-settings.conf` (or another vendor sysctl.d file) actually sets `vm.page-cluster=0` and `vm.vfs_cache_pressure=50` — i.e. the profile's removal leaves the same effective values. If the vendor default differs (e.g. page-cluster=3, vfs_cache_pressure=100), the removal is a SILENT behavior change for zram readahead and cache reclaim — flag it. List the exact vendor values.
- **zram pair (remaining):** confirm the running kernel accepts swappiness > 100; is 150 gratuitous on 128 GB RAM, or does it help large-alloc/LLM reclaim? List which vendor 70-cachyos values the priority-95 file still overrides (now just swappiness/compaction/max_map_count) and whether justified.
- zswap.enabled=0: confirm CachyOS uses zram (not zswap) by default — no double-compression conflict.
- NVMe scheduler none: confirm best practice vs mq-deadline/kyber on this kernel; evaluate nr_requests and read_ahead_kb. Confirm the rename to `99-ry-perf.rules` correctly sorts after vendor `60-ioschedulers.rules` so the profile wins.
- fstab noatime,lazytime,commit=10: confirm noatime and lazytime coexist; weigh commit=10 durability (esp. now that `fsck.mode=force` runs every boot §3); confirm fstrim.timer over continuous discard.
- vm.max_map_count near 2^31: confirm appropriate for Proton/anti-cheat. compaction_proactiveness=0: confirm right for gaming + large unified allocs.
- Confirm not enabling systemd-oomd (kernel OOM + zram) is right on 128 GB.
- Sources: docs.kernel.org (block, sysctl/vm), wiki.archlinux.org/Zram + SSD + Ext4, wiki.cachyos.org (zram + sysctl defaults).

### 8. Network and latency

sysctl net: default_qdisc=fq, netdev_budget=600, netdev_budget_usecs=5000, tcp_congestion_control=bbr, tcp_notsent_lowat=16384, tcp_slow_start_after_idle=0. NM: wifi.backend=wpa_supplicant (NM default; iwd opt-in via NM_WIFI_BACKEND), wifi.powersave=2 (off), logging WARN. **NEW: `/etc/modprobe.d/60-ry-mt7925e.conf` → `options mt7925e disable_aspm=1`.** resolved: MulticastDNS=no, LLMNR=no, DNSOverTLS=no, DNSSEC=allow-downgrade (plaintext; diverges from CachyOS DoH default). regdom: COUNTRY=US fixed (`/etc/iw-regdomain`). Masked: NetworkManager-wait-online, modemmanager. Enabled: NetworkManager.

- **mt7925e disable_aspm=1 (NEW drop-in):** confirm disabling PCIe ASPM on MT7925 is the current correct mitigation for coredump/BT-reconnect/assoc-fail, and whether it is still needed or an upstream mt76 fix has landed (→ if landed, prefer a kernel/firmware floor and flag the drop-in as removable). Confirm it does not fight `pcie_aspm.policy=performance` (§3) — are both saying "no ASPM" on this device, making one redundant? Note the drop-in is explicitly symptomatic ("drop if upstream resolves").
- **NM backend wpa_supplicant:** verify current Arch/CachyOS guidance for MT7925/mt76 — is wpa_supplicant the more stable backend today, or has iwd matured to parity? Confirm wifi.powersave=2 alone fully disables the mt76 software power-save latency issue under wpa_supplicant, and that backend + the new ASPM drop-in + btusb.enable_autosuspend=n (§3) together close the MT7925 stability gap. Confirm no dangling iwd reference.
- bbr + fq: confirm still recommended; clarify BBRv3 status in current kernels.
- Dual 10 GbE: validate netdev_budget/usecs for 10G; assess tcp_rmem/wmem or NIC ring tuning for line-rate (RTL8127, §11).
- tcp_notsent_lowat=16384, tcp_slow_start_after_idle=0: confirm rationale holds.
- mDNS off (consistent with nftables drop §10); plaintext DNS reverting DoH — confirm CachyOS ships an encrypted-DNS default this overrides; flag the privacy reduction (§10).
- regdom US fixed: confirm correct max TX power and channel set for MT7925 on current cfg80211/wireless-regdb. Flag if 6 GHz Wi-Fi 7 needs a specific AFC posture beyond a country code; note non-US deployers must hand-edit COUNTRY.
- Sources: docs.kernel.org/networking (bbr, fq, tcp), wiki.archlinux.org/Sysctl + NetworkManager + Wireless/MediaTek, git.kernel.org wireless-regdb + mt76.

### 9. systemd units

Mask (10): ananicy-cpp, power-profiles-daemon, NetworkManager-wait-online, ufw, modemmanager, sleep/suspend/hibernate/hybrid-sleep/suspend-then-hibernate targets. Enable (5): fstrim.timer, NetworkManager, cpupower, nftables, bluetooth. Not enabled: systemd-oomd (intentional), NetworkManager-dispatcher (socket-activated), rtkit-daemon (assess §2). iwd.service untouched (opt-in). ufw flushed after nftables live.

- For each mask, confirm safe and beneficial on CachyOS: ananicy-cpp + ppd (§6); modemmanager (no cellular HW); sleep/suspend masked = no suspend at all (matches an always-on mini-PC, breaks nothing in logind).
- bluetooth.service enabled (with main.conf §12): confirm AutoEnable=true posture and BlueZ key currency.
- Confirm nftables is enabled as the firewall and that ufw-flush-then-mask leaves no unfirewalled window (the script already guards: if nft not confirmed live, ufw mask is skipped to preserve coverage — confirm this handoff is correct).
- fstrim.timer vs continuous discard; cpupower vs CachyOS freq management; confirm oomd stays disabled and dispatcher stays socket-activated.
- logind Handle*Key=ignore incl LongPress: confirm intended, no lockout risk.
- Sources: man.archlinux.org (systemd.unit, logind.conf), wiki.archlinux.org (Bluetooth), wiki.cachyos.org.

### 10. Security and safety (cross-cutting)

nftables default-deny-inbound (ufw masked): input policy drop, ct established/related accept, lo accept, ct invalid drop, ICMPv6 NDP/PMTUD + echo-request accept, IPv4 ICMP diagnostics-only (no inbound ping), forward drop, output accept. **NEW: `RY_REMOTE_PLAY_PORTS` gate (default false)** — when true, appends `tcp dport { 47984, 47989, 48010, 27036 }` and `udp dport { 47998-48010, 27031-27036 }`. iommu=pt (AMD-Vi on). clearcpuid=514 (UMIP off). split_lock_detect=off.

- **RY_REMOTE_PLAY_PORTS port set (NEW):** validate the exact ports against current Sunshine/Moonlight + Steam Remote Play requirements. Confirm `47984/47989/48010` (Sunshine HTTPS/HTTP/RTSP), `27036` (Steam), and the UDP ranges are correct and complete for current versions — flag any missing port (e.g. Sunshine video/control/audio/mic UDP) or any stale one. Confirm default-OFF is the right posture and that enabling appends cleanly without weakening the established/related baseline.
- Validate the nftables shape is minimal-but-sufficient on a dual-10 GbE LAN.
- IPv4 ping dropped, IPv6 reachable: confirm the intended asymmetric LAN posture.
- ICMPv6 NDP/PMTUD accept must be present (static `_vss_nft` hard-fails if nd-neighbor-solicit is missing; live check warn-only). Treat the static rule as the gate.
- iommu=pt net positive vs prior amd_iommu=off; confirm AMD-Vi genuinely enabled, record as a closed reduction.
- clearcpuid=514 is the open reduction; quantify exposure vs the umip_printk stutter prevented (§3); list first in the security delta.
- split_lock_detect=off: quantify residual exposure.
- Produce the ordered security-delta subsection (above), now including the remote-play opt-in at position 4.
- Sources: wiki.archlinux.org (nftables, Security, IOMMU), docs.kernel.org (split lock, UMIP), github Sunshine/Moonlight (port reference).

### 11. Known issues and DKMS currency

MES page faults → resolved (MES 0x86; linux-firmware + mkinitcpio-firmware). RTL8127 throughput + suspend/shutdown hang → resolved in-tree r8169 (`f24f7b2f3af9` + `ae1737e7339b`); **now enforced as KERNEL_MIN 6.18**; no DKMS. MT7925 panics/deauth/coredump → mitigated via `disable_aspm=1` drop-in (§8) + `btusb.enable_autosuspend=n` (§3) + wpa_supplicant backend; upstream status open. Strix Halo ACP → open (HDMI/USB audio unaffected; ACP70 has no matching ASoC machine driver — internal mic undetected, flagged by `_vss_known_benign` §12). Install pacman-only.

- Re-confirm the resolved claims: the MES 0x86 blob version carried by the installed linux-firmware + mkinitcpio-firmware, and the minimum kernel containing BOTH r8169 commits — verify it is exactly 6.18 (the new floor) or state the true value. Give exact landing versions.
- MT7925: with three mitigations now stacked (ASPM-disable drop-in, btusb autosuspend off, wpa_supplicant), assess whether the gap is closed or whether a kernel/firmware floor is still the better path. Check current upstream mt76 status for the coredump/assoc-fail bugs — has a fix landed that would let the symptomatic drop-in be dropped?
- ACP/internal mic: confirm the ACP70 machine-driver gap is still open upstream and that the only fix is a kernel board-ID quirk (i.e. nothing the profile can ship). Confirm reporting the board model upstream is the correct action.
- Recommend a kernel/firmware floor over DKMS for any landed fix; confirm any suggested DKMS still builds.
- Sources: gitlab.freedesktop.org/drm/amd, git.kernel.org linux-firmware + r8169 (`f24f7b2f3af9`, `ae1737e7339b`) + mt76, bugzilla.kernel.org, discuss.cachyos.org.

### 12. MangoHud, Bluetooth, and hygiene

**MangoHud.conf (18 directives, 0600, fps/frametime first):** horizontal, legacy_layout=0, position=top-left, **toggle_hud=Shift_R+F12 (restored v7.71.3)**, fps, frametime, frame_timing, gpu_stats, gpu_core_clock, gpu_temp, **gpu_power (restored v7.71.3)**, cpu_stats, cpu_mhz, vram, ram, font_size=20, **text_outline (restored v7.71.3)**, background_alpha=0.4. Still absent vs older revisions: cpu_temp, fps_metrics. Enabled via MANGOHUD=1. bluetooth main.conf: FastConnectable=true, AutoEnable=true, ReconnectAttempts=3 (no explicit ReconnectIntervals — BlueZ default backoff). baloofilerc: Indexing-Enabled=false.

- **MangoHud (gpu_power + toggle_hud + text_outline restored):** the prior prompt flagged their removal as a possible TUNE-to-restore; v7.75.1 restored them. Confirm all 18 directives are valid for current MangoHud (do not invent). With `gpu_power` back, confirm it populates from amdgpu sensors on gfx1151 under Wayland (the most decision-relevant readout on a 140 W shared-package APU). `cpu_temp` remains absent — assess whether it should also return (CPU thermal is the other half of the shared-package picture). Confirm `toggle_hud=Shift_R+F12` is a valid bind on current MangoHud. Establish the real minimum MangoHud + kernel for the amdgpu sensors in use.
- Confirm gpu_temp/gpu_core_clock/vram/cpu_mhz populate from amdgpu sensors under Wayland on this iGPU; confirm vram + ram is the right unified-memory representation.
- Overhead: confirm near-zero, no conflict with the gamescope overlay or GameMode.
- **Bluetooth main.conf:** confirm BlueZ keys/sections current; ReconnectAttempts=3 + BlueZ default reconnect backoff sane for paired audio sinks; AutoEnable=true fixes adapter-off-at-boot; FastConnectable=true no meaningful downside on an always-on desktop. Cross-check against the cmdline BT changes: `btusb.enable_autosuspend=n` (§3) — confirm the kernel-level autosuspend-off and the BlueZ-level reconnect policy are complementary, not redundant.
- baloofilerc Indexing-Enabled=false: confirm [Basic Settings]/Indexing-Enabled current for the installed Baloo.
- **Known-benign verify surface (`_vss_known_benign`, NEW v7.74.2):** confirm the 5 INFO conditions are correctly characterized as benign on this exact hardware and that none mask a real fault: (1) masked ModemManager D-Bus activation noise, (2) ACP70 no-matching-ASoC-machine-driver (mic undetected), (3) boltd unknown-NHI-PCI-id (USB4/TB UID-stability skipped), (4) no battery/backlight sysfs (desktop mini-PC), (5) USB-mic volume-curve quirk. Validate each is genuinely cosmetic/expected, not a swallowed regression.
- **Naming:** every human-facing string reads "GTR9 Pro" but the internal PROFILE_NAME and function names carry `gtr_pro` (FIX, Low/Low) — do not change if it breaks function-name refs or log fields; cosmetic.
- Sources: github flightlessmango/MangoHud (config ref, version tags), wiki.archlinux.org/MangoHud + Bluetooth, man.archlinux.org bluetooth main.conf, KDE Baloo docs.

## Scope and non-goals

- Recommendations only — do not emit a modified script.
- Out of scope: dotfiles, shells, editors, secrets, backups, multi-user, non-CachyOS, laptops, UKI.
- Per-game Proton tuning is secondary; prioritize system-wide config.
- **Reinstatement rule update:** PROTON_FSR4_RDNA3_UPGRADE and MangoHud gpu_power/text_outline/toggle_hud have been RE-ADDED in this revision — they are no longer "deliberately removed." Evaluate them as live config (KEEP if upstream-valid, FIX-to-remove if inert/harmful). Items still deliberately removed (do not recommend reinstating unless upstream directly contradicts the rationale, then flag not FIX): amdgpu.ppfeaturemask, `--country` flag, TTM/GTT cap, RADV drirc, MangoHud cpu_temp/fps_metrics, vm.page-cluster/vm.vfs_cache_pressure (vendor-provided).
- IOMMU special case: amd_iommu=off is removed; current is iommu=pt — treat changes under the reinstatement rule, record as a restored mitigation.
- Governor/EPP special case: performance→powersave + performance→balance_performance was deliberate (EPP-honoring under active mode) — do not flag powersave as a regression without proving the performance governor would override the EPP hint.
- **GPU_DPM_LEVEL special case (NEW):** `high`→`auto` was deliberate (v7.71.0, to stop pinning SCLK and stealing Zen 5 boost). Do not flag `auto` as a GPU-perf regression without proving `high` materially improves frametime/1%-lows on gfx1151 without costing CPU boost on the shared 140 W package.

---

# Deep-pass appendix (exact generated bodies + full verify surface)

The §1–§12 investigation is value-level. This appendix is artifact-level: the exact
strings the script writes, the complete verify subsystem, and the install-phase model.
Validate the **rendered file content**, not a paraphrase. Every block below is quoted
from the generator functions in `ry-install.fish` v7.75.1 — treat each as the literal
deployed file (UUIDs/joins resolved at runtime).

## A. Install-phase model (`_RY_PHASE_NAMES`)

Six ordered phases; recommendations must respect this sequence (a fix valid in
isolation can be wrong if it reorders a phase):

```
1 Preflight     _install_preflight     — _ir_* gates (counts, keys, kernel floor, post-hooks, root UUID)
2 Packages      _install_packages      — pacman -Syu PKGS_ADD; chwd Vulkan; PKGS_DEL handled in Services
3 Configuration _install_system_files  — render+deploy all 18 managed files (atomic tmp+rename)
4 Services      _install_configure_services — fstab rewrite + resolved + PKGS_DEL + mask + iwd handoff + enable + regdom + nft/ufw handoff
5 Boot          _install_rebuild_boot  — mkinitcpio -P + sdboot-manage update (gated on boot-critical writes)
6 Finalize      _install_finalize      — user daemon-reload + pacman cache trim + NetworkManager restart
```

- Confirm the **firewall handoff lives in Phase 4** (nftables made live *before* ufw is
  flushed/masked) and that boot-critical regeneration (Phase 5) only fires when one of
  `_RY_BOOT_CRITICAL_DSTS` actually changed. Flag any recommendation that would move a
  cmdline/mkinitcpio change outside the Phase-5 gate.
- `_RY_BOOT_CRITICAL_DSTS` (4): `/boot/loader/loader.conf`, `/etc/kernel/cmdline`,
  `/etc/sdboot-manage.conf`, `/etc/mkinitcpio.conf`. `_RY_BACKUP_TARGETS` (2, `.ry.bak`):
  `/boot/loader/loader.conf`, `/etc/mkinitcpio.conf`. Confirm the backup set is
  sufficient (note `/etc/kernel/cmdline` is regenerable from `KERNEL_PARAMS`, but
  `loader.conf`/`mkinitcpio.conf` may carry pre-existing user state — is 2 enough?).
- `_RY_POST_HOOKS` (18): each managed-file glob maps to a `_post_<tag>` reload handler
  (`boot`, `cmdline`, `resolved`, `logind`, `nmdispatch`, `nm`, `regdom`, `bluetooth`,
  `nft`, `cpupower`, `sysctl`, `udev`, `modprobe`, `envd`, `baloo`, `mangohud`).
  `_ir_validate_post_hooks` refuses deploy if any tag lacks a handler. Confirm
  `--install-file <path>` of any single managed file triggers the correct reload and
  that the modprobe/udev/cmdline handlers correctly defer to reboot.

## B. Exact rendered file bodies (validate content, not summary)

### B1. `/etc/kernel/cmdline` + `/etc/sdboot-manage.conf`

```
rw root=UUID=<_ROOT_UUID> 8250.nr_uarts=0 amd_pstate=active btusb.enable_autosuspend=n clearcpuid=514 fsck.mode=force fsck.repair=yes iommu=pt nowatchdog nvme_core.default_ps_max_latency_us=0 pcie_aspm.policy=performance processor.max_cstate=1 quiet split_lock_detect=off tsc=reliable usbcore.autosuspend=-1 zswap.enabled=0
```
sdboot-manage.conf: `LINUX_OPTIONS` = same KERNEL_PARAMS join; `LINUX_FALLBACK_OPTIONS="quiet"`; `DEFAULT_ENTRY="manual"`; `REMOVE_EXISTING="yes"`; `OVERWRITE_EXISTING="yes"`; `REMOVE_OBSOLETE="yes"`.

- **Cross-file consistency:** KERNEL_PARAMS is rendered into BOTH `/etc/kernel/cmdline`
  (kernel-install path) and `sdboot-manage.conf` `LINUX_OPTIONS` (sdboot-manage path).
  Confirm both bootloader-management paths are not simultaneously active in a way that
  double-writes or conflicts entries; state which one CachyOS actually drives and
  whether maintaining both is redundant or a divergence risk.
- `LINUX_FALLBACK_OPTIONS="quiet"` strips ALL performance/safety params from the
  fallback entry (keeps only `quiet`). Confirm a fallback boot with none of
  `amd_pstate`/`iommu=pt`/`fsck.*` is the intended recovery posture (it is arguably
  correct — minimal fallback — but verify `fsck.mode=force` being absent from fallback
  doesn't matter).

### B2. `/boot/loader/loader.conf`
```
default @saved
timeout 0
console-mode keep
editor no
```
- `default @saved` + `timeout 0`: confirm `@saved` resolves on systemd-boot with
  timeout 0 (no menu) — does a failed boot still let the user reach the menu, or does
  timeout 0 + editor no create a recovery dead-end requiring external media? Flag the
  recovery-ergonomics trade.

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
- **Rule-order correctness:** `ct state invalid drop` sits AFTER `established,related
  accept` and `lo accept`. Confirm this ordering cannot drop a valid loopback or
  established packet (it should not — established is matched first — but verify against
  current nft semantics).
- **ICMPv6 set (8 types):** confirm the set is minimal-but-complete for NDP + PMTUD +
  diagnostics: nd-neighbor-solicit/advert + nd-router-advert/solicit (NDP),
  packet-too-big (PMTUDv6, load-bearing), time-exceeded/parameter-problem (diag),
  echo-request (IPv6 ping allowed). Flag any missing type that breaks SLAAC or MLD
  (e.g. `mld-listener-query` / `nd-redirect` are absent — confirm that is intended on a
  LAN with a single router).
- **IPv4 ICMP (4 types):** echo-reply/destination-unreachable/time-exceeded/
  parameter-problem accepted; echo-request (inbound ping) deliberately absent. Confirm
  `destination-unreachable` accept preserves IPv4 PMTUD (frag-needed). This is the
  asymmetric posture (IPv4 unpingable, IPv6 pingable) — confirm intended.
- **`flush ruleset` at top:** confirm flushing the entire ruleset (not just the inet
  filter table) is safe — it wipes any other table (nat, mangle) a user or another tool
  may have added. Flag if CachyOS or docker/libvirt would lose rules.
- **No rate-limiting on ICMP/new-conn:** confirm the absence of `limit rate` on
  echo-request is acceptable on a trusted LAN (no ICMP flood mitigation).
- The static verify `_vss_nft` hard-fails if `nd-neighbor-solicit` is absent — treat
  that single rule as the IPv6 break-glass gate.

### B4. udev `99-ry-perf.rules` (exact — 3 rules)
```
ACTION=="add|change", KERNEL=="nvme[0-9]*n[0-9]*", ENV{DEVTYPE}=="disk", ATTR{queue/scheduler}="none"
ACTION=="add|change", SUBSYSTEM=="cpu", DEVPATH=="*/cpufreq", ATTR{cpufreq/energy_performance_preference}="balance_performance"
ACTION=="add", KERNEL=="card[0-9]", SUBSYSTEM=="drm", DRIVERS=="amdgpu", ATTR{device/power_dpm_force_performance_level}="auto"
```
- **NVMe rule `add|change`** (re-asserts on change uevents) vs **GPU rule `add` only**
  (one-shot at enumeration). Confirm the asymmetry is intended: EPP re-asserts after
  AC/DC transitions (add|change) but DPM=auto is set once. With DPM now `auto`, the
  one-shot is harmless; if a reviewer recommends returning to `high`, note the `add`-only
  rule would NOT re-assert after a GPU reset — flag that as a reason the old `high` was
  fragile.
- `KERNEL=="card[0-9]"` matches card0–card9 only (single-digit). Confirm no multi-GPU/
  render-node (`renderD*`) concern on this single-iGPU host.
- Filename `99-` sorts after vendor `60-ioschedulers.rules` (CachyOS kyber default) —
  confirm 99 actually wins (last-matching-rule semantics for ATTR assignment).

### B5. `/etc/systemd/resolved.conf.d/99-cachyos-resolved.conf`
```
[Resolve]
MulticastDNS=no
LLMNR=no
DNSOverTLS=no
DNSSEC=allow-downgrade
```
- This drop-in is `99-` and named `cachyos-resolved` — confirm it overrides, not
  collides with, any vendor `cachyos`-shipped resolved drop-in of the same basename
  (same filename = replace, not merge). If CachyOS ships its OWN
  `99-cachyos-resolved.conf` (DoH default), this file *replaces* it — confirm that is
  the intended override mechanism and not an accidental basename clash.

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
- Confirm `LogLevelMax=notice` on the dispatcher drop-in is the correct journald-noise
  fix (the comment notes `StandardError=null` is ineffective because dispatcher logs via
  journald) — validate that reasoning against current NM-dispatcher behavior.

### B7. `/etc/iw-regdomain` + persistence path
```
COUNTRY=US
```
- **Persistence mechanism (name it precisely):** the runtime verify states regdom
  persists via `/etc/iw-regdomain → cachyos-iw-set-regdomain`. Confirm
  `cachyos-iw-set-regdomain` is the actual CachyOS unit/script that reads
  `/etc/iw-regdomain` and runs `iw reg set` at boot — and that it still exists in
  current CachyOS (this is the load-bearing assumption; if CachyOS dropped it, the
  regdom file is inert and the profile must switch to `wireless-regdb`/`crda` or a
  systemd unit). This is the single most version-fragile external dependency in the
  profile.

### B8. `/etc/bluetooth/main.conf` + `/etc/default/cpupower-service.conf`
```
[General]
FastConnectable=true
[Policy]
AutoEnable=true
ReconnectAttempts=3
```
cpupower-service.conf: `GOVERNOR='powersave'` (sourced by `/usr/lib/systemd/scripts/cpupower`).
- Confirm `cpupower.service` on CachyOS actually sources `/etc/default/cpupower-service.conf`
  for `GOVERNOR` (path + var name). If CachyOS uses a different cpupower config path or
  unit, this file is inert and the governor is unset by the profile (the udev EPP rule
  still applies, but the governor would fall to kernel default). Validate the path.

### B9. `~/.config/MangoHud/MangoHud.conf` (exact 18 directives, ordered)
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
cpu_mhz
vram
ram
font_size=20
text_outline
background_alpha=0.4
```
- Confirm `frame_timing` (the frametime graph) vs `frametime` (numeric) are both valid
  current keys and not redundant. Confirm `toggle_hud=Shift_R+F12` keybind syntax is
  current. `cpu_temp` absent while `gpu_temp`/`gpu_power` present — flag the asymmetry
  on a shared-package APU (the §12 TUNE-to-restore-cpu_temp question).

## C. Full verify subsystem (`--verify`) — surfaces the VERIFY block omits

The top VERIFY block is the user-facing command set; the script's actual `--verify`
runs three check families. Recommendations that change a value MUST state which verify
sub asserts it (and whether hard-fail or warn):

### C1. Static-system subs (`_vss_*`) — file content on disk
`_vss_ntsync_modules` (ntsync state + autoload confs) · `_vss_logind` · `_vss_nmdispatch`
· `_vss_nm` · `_vss_sysctl` (key=value) · `_vss_regdom` · `_vss_bluetooth` · `_vss_udev`
(all 3 rules) · `_vss_nft` (**hard-fail on missing nd-neighbor-solicit**) · `_vss_modprobe`
(mt7925e disable_aspm=1) · `_vss_known_benign` (5 INFO, never fails).

### C2. Runtime-env subs (`_vre_*`) — live kernel/daemon state
`_vre_envvars` (ENV_VARS via `systemctl --user show-environment`) · `_vre_sysctl_runtime`
(`/proc/sys`) · `_vre_tcp` (tcp_bbr module) · **`_vre_zram`** (zram service + active swap
device) · **`_vre_fstab`** (asserts ext4 has `noatime,lazytime,commit=10`) · `_vre_ntsync`
(state dispatch) · `_vre_regdom` (`iw reg get`).

- The prompt's VERIFY block omits `_vre_zram` and `_vre_fstab` — add live checks:
  `swapon --show` / `zramctl` (zram active, expected swap device) and
  `findmnt -no OPTIONS /` (noatime,lazytime,commit=10). Confirm what `_vre_zram` treats
  as PASS vs WARN given ZRAM is "out-of-scope advisory" (masked-zram → WARN, no-swap →
  WARN) — the profile does not manage zram but does verify it. Flag the
  manages-nothing-but-asserts-something tension.

### C3. Runtime-session subs (`_vrs_*`) — installed-file perms + Vulkan
`_vrs_nm_perms` (**system-connections 0600 root:root**) · `_vrs_vfat_skip` (skips perm
check on vfat/undetermined boot) · `_vrs_installed_file_perms` (system/service/user file
modes) · `_vrs_parent_dirs` (parent-dir ownership) · `_vrs_vulkan` (vulkan-radeon +
lib32-vulkan-radeon present; DXVK/VKD3D-Proton dependency).

- Add to VERIFY block: `stat -c '%a %U:%G' /etc/NetworkManager/system-connections/*`
  (expect 0600 root:root), `vulkaninfo` ICD presence, and a perms spot-check on the
  managed files (configs 0644, MangoHud/baloo user-owned). Confirm the `_vrs_vfat_skip`
  carve-out (perms unverifiable on a vfat `/boot`) is correct and not hiding a real
  perms regression on `/boot/loader/loader.conf`.

## D. fstab rewrite (`_install_fstab_opts`) — normalization, not just append

The rewrite does more than add tokens. Validate the normalization logic:
- Adds `noatime,lazytime,commit=10` to ext4 entries (field 4 only).
- **Strips conflicting tokens:** `relatime`, `atime`, `strictatime` are detected as
  contradicting `noatime` (verify `_vre_fstab` fails the entry if any coexist with
  noatime).
- **Rewrites an existing `commit=N`** to `commit=10` (captures `commit=([0-9]+)` and
  replaces) rather than appending a duplicate.
- Operates via an awk pass that PRESERVES already-correct lines
  (`$4 ~ noatime && $4 ~ lazytime && $4 ~ commit=10 { print; next }`).

Items to confirm: (a) the rewrite touches ONLY ext4 (not the vfat ESP, not any
btrfs/xfs); (b) it is idempotent (re-run is a no-op on already-correct fstab); (c) it
writes atomically (tmp+rename) so a crash mid-rewrite cannot truncate fstab — this is a
boot-critical file, confirm the atomic-write guarantee; (d) `commit=10` durability
trade is acceptable given `fsck.mode=force` now runs every boot (§3) — do the two
interact (forced fsck offsets the wider commit window)?

## E. Preflight gate ordering (`_install_preflight` / `_ir_*`)

Order matters for exit-code semantics. Confirm the gate sequence and that each maps to a
distinct exit code:
- `_ir_validate_counts` (EXIT_PREFLIGHT 3) — all 21 array counts.
- `_ir_validate_keys` (3) — embedded scalar domains (true|false, yes|no, int, ISO-3166).
- `_ir_validate_kernel_floor` (3) — kernel ≥ 6.18 (override
  `RY_INSTALL_SKIP_KERNEL_FLOOR_CHECK=1`).
- `_ir_validate_post_hooks` (3) — every `_RY_POST_HOOKS` tag has a handler.
- `_ir_resolve_root_uuid` → `EXIT_GEN_NOUUID 12` if cmdline render finds no UUID.
- Hardware gate (CPU match, override `RY_INSTALL_SKIP_HARDWARE_CHECK=1`).
- Generator exit codes: `EXIT_GEN_NOFN 11` (no content fn), `EXIT_GEN_NOUUID 12`,
  `EXIT_GEN_SYSCTL 13` (sysctl count mismatch in `_content_*sysctl`).

Confirm: (a) counts/keys/floor run BEFORE any disk write (they do — preflight precedes
Configuration); (b) the three skip-override env vars are the only documented bypasses and
each is scoped (kernel-floor and hardware only — counts/keys cannot be bypassed); (c)
`PACTREE_TIMEOUT_S=60` and `BOOT_SPACE_CRIT=200`/`BOOT_SPACE_WARN=500`/
`ROOT_AVAIL_CRIT=2`/`ROOT_AVAIL_WARN=5` (MB/GB disk thresholds) are sane gates for a
dual-NVMe host — are the boot-partition thresholds (200/500 MB free) right for a
systemd-boot ESP holding multiple kernels + fallback?

## F. Deeper-pass investigation deltas (new actionable items)

1. **Dual bootloader-management surface (B1):** resolve whether `/etc/kernel/cmdline`
   (kernel-install) and `sdboot-manage.conf` `LINUX_OPTIONS` are both consumed —
   redundant double-source of KERNEL_PARAMS is a drift risk. Determine the single
   authoritative path on current CachyOS.
2. **`cachyos-iw-set-regdomain` existence (B7):** verify this external CachyOS unit
   still exists and reads `/etc/iw-regdomain`. Highest version-fragility in the profile.
3. **`cpupower.service` config path (B8):** verify CachyOS sources
   `/etc/default/cpupower-service.conf` `GOVERNOR`. If not, the governor is profile-unset.
4. **`99-cachyos-*` basename overrides (B5/B6):** confirm same-filename replace (not
   merge) is the intended override of vendor resolved/NM drop-ins, and not an accidental
   clash that a CachyOS update would overwrite.
5. **`flush ruleset` blast radius (B3):** confirm wiping all nft tables is safe vs
   docker/libvirt/podman nat tables on this host.
6. **`_vre_zram` / `_vre_fstab` coverage (C2):** add both to the user VERIFY block;
   reconcile "zram out-of-scope" with "zram asserted in --verify".
7. **`_vrs_nm_perms` 0600 root:root (C3):** confirm current NM still requires/writes
   system-connections at 0600 and the assert is not stale.
8. **fstab atomic-write + idempotency (D):** confirm tmp+rename on the boot-critical
   fstab and no-op on re-run; reconcile `commit=10` with forced-fsck-every-boot.
9. **`LINUX_FALLBACK_OPTIONS="quiet"` recovery posture (B1):** confirm a param-stripped
   fallback entry is the intended recovery boot and `timeout 0`+`editor no` (B2) is not a
   recovery dead-end.
10. **`card[0-9]` single-digit match (B4):** confirm no >9-card / renderD edge on this
    single-iGPU host (cosmetic, but a correctness note for the udev match).

---

# Deepest-pass appendix (§G–§L) — robustness & correctness audit surface

§1–§12 audit *what the profile configures*; §A–§F audit *what the script writes and
asserts*. This final layer audits *whether the installer is safe to run at all* — the
atomic-write, locking, privilege, rollback, and signal machinery. These are not tuning
questions; they are correctness questions. A reviewer must confirm each guarantee holds
on current fish (3.x floor) / CachyOS, and flag any TOCTOU, fail-open, or
partial-write window. Every mechanism below is quoted from `ry-install.fish` v7.75.1.

## G. Atomic-write guarantees (`_awf_*`)

Write path per managed file: `_awf_render_to_tmp` → `_awf_symlink_check` →
`_awf_finalize_mv`; boot/backup targets add `_awf_make_backup` (pre) +
`_awf_postwrite_verify_restore` (post).

Mechanisms to confirm:
- **tee-to-tmp with `$pipestatus`:** content generator piped into `_as $use_sudo tee --
  "$tmpfile"`; `$pipestatus[1]` (generator rc) and `$pipestatus[2]` (tee rc) checked
  separately, mapping generator failures to `EXIT_GEN_NOFN/NOUUID/SYSCTL`. Confirm fish
  `$pipestatus` semantics still distinguish the two stages reliably and that a
  generator returning non-zero never leaves a partial tmpfile promoted.
- **Post-write symlink-swap probe (`_awf_symlink_check`):** after writing tmp, re-tests
  whether tmp was replaced by a symlink (rc 0/1/2 = symlink/not/sudo-lapse), aborting on
  swap. This is a TOCTOU defense against an attacker swapping the tmp path between write
  and rename. Confirm the probe closes the window (is there still a gap between the
  symlink check and the `mv -T`?).
- **`mv -T` atomic rename (`_awf_finalize_mv`):** chmod tmp → re-assert `sudo -n true`
  (credential-lapse guard) → `mv -T -- tmp dst`. Confirm `mv -T` is atomic on the same
  filesystem (tmp is created in dst's parent via `_tmp_dir`/dst-parent so the rename is
  same-FS — verify this holds for `/boot` on vfat, where rename atomicity differs).
- **Post-write byte re-read + restore (`_awf_postwrite_verify_restore`):** re-runs the
  generator, reads installed bytes (`_installed_bytes`, tri-state rc), compares; on
  mismatch restores `.ry.bak` via `mv -T`. Confirm: (a) the generator is deterministic
  so a re-run equals the original (the sysctl generator has side-effects on
  `_RY_SYSCTL_BAD_ENTRIES` — the preflight already refuses `_RY_BACKUP_TARGETS` members
  that use side-effecting generators; confirm that guard is complete); (b) restore via
  `mv -T` of the backup is itself atomic; (c) `string collect --no-trim-newlines
  --allow-empty` correctly preserves trailing-newline-sensitive comparisons.
- **Backup only for `_RY_BACKUP_TARGETS` (2):** loader.conf + mkinitcpio.conf. Every
  other managed file relies solely on atomic-write (no .bak). Confirm a post-write
  mismatch on a NON-backup file (e.g. nftables.conf) is detected but cannot be
  auto-restored — is that the right risk posture for the firewall ruleset?

Actionable: confirm the symlink-check→mv window is non-exploitable; confirm `/boot`
vfat rename atomicity; confirm generator determinism for all post-write-verified files.

## H. Instance lock & PID-recycle TOCTOU (`_acquire_lock*`, `_lock_pid_started_after`)

Lock = atomic `mkdir "$LOCK_DIR"` (umask 0077, chmod 700) + pidfile written via
`mktemp`+`mv -Tf` (chmod 600). Stale reclaim is bounded (3 attempts) and fail-closed.

Mechanisms to confirm:
- **Atomic mkdir as the lock primitive:** `mkdir` is the atomicity guarantee (fails if
  dir exists). Confirm this is race-free vs a competing instance on the same host.
- **PID-recycle detection (`_lock_pid_started_after`):** reads `/proc/PID/stat` field 22
  (starttime in ticks), adds `/proc/stat` btime, divides by USER_HZ (`getconf CLK_TCK`,
  with **CONFIG_HZ recovery from `/proc/config.gz`** if getconf absent), and reclaims
  ONLY if the holder's start time is provably > pidfile mtime + 2s. **Fail-closed**:
  any unparseable field → treat as live, refuse reclaim. Confirm: (a) the
  `string replace -r '^.*\) '` correctly handles a `comm` field containing `) ` (the
  classic `/proc/stat` parsing trap); (b) field-22 indexing survives a process whose
  comm embeds spaces/parens; (c) the +2s mtime slack is sufficient against clock
  granularity but tight enough to catch a genuinely recycled PID.
- **Re-read-before-rm TOCTOU guard:** before `rm -rf` of a stale lock, the pidfile is
  re-read and compared to the value seen at decision time; if it changed mid-pass,
  abort (another instance is active). Confirm this closes the reclaim race.
- **Symlink refusal:** if `$LOCK_DIR` is a symlink, reclaim is refused. `rm -rf
  --preserve-root` used. Confirm no path-traversal via a crafted lock dir.
- **kill -0 EPERM → /proc branch:** unsignalable-but-alive PIDs (different UID) are
  detected via `/proc/PID` presence and NOT reclaimed. Confirm correct.

Actionable: confirm the `/proc/stat` comm-parsing is robust; confirm fail-closed on
every USER_HZ/btime/starttime read failure; confirm the re-read-before-rm window is
genuinely closed on current kernels.

## I. Privilege handling (`_as`, `_run`, `_is_symlink`, `_installed_bytes`)

- **`_as use_sudo` / `sudo -n` everywhere:** all privileged ops use non-interactive
  `sudo -n`; credential lapse is re-checked immediately before each critical write
  (`mv`, mkinitcpio revert). Confirm there is NO interactive sudo prompt mid-run (which
  would hang an unattended install) and that a mid-run credential expiry fails safe
  (aborts the file, does not partial-write).
- **Tri-state rc 0/1/2 (drift vs sudo-lapse):** `_is_symlink`, `_installed_bytes`,
  `_ry_content_bytes` all return 2 specifically for sudo-cache-lapse so the caller can
  distinguish "file differs" from "couldn't read due to expired sudo." Confirm every
  caller branches on rc 2 (a collapse of 2→1 would misreport a sudo lapse as a drift
  failure). This is a subtle correctness invariant worth spot-checking across callers.
- **`_run` timeout enforcement:** `_run` wraps commands with logging + stdout/stderr
  capture + timeout. Confirm pacman (`_ip_pacman_invoke`) and mkinitcpio are not killed
  by a too-short timeout on a slow mirror / large initramfs (these are the longest ops).
  `PACTREE_TIMEOUT_S=60` governs pactree; confirm pacman/mkinitcpio have no fatal cap.

Actionable: audit every rc-2 caller for the drift-vs-lapse distinction; confirm no
interactive-sudo hang path; confirm long-op timeouts are non-fatal.

## J. Boot-wipe gate & boot-critical rollback (`_irb_taint_gate`, `_install_rebuild_boot`)

The single most dangerous operation: `SDBOOT_REMOVE_EXISTING=yes` wipes foreign BLS
entries. The gate sequencing:
1. `_irb_taint_gate` → `_check_boot_taint_gate`: if `_RY_BOOT_TAINTED=true` OR the
   mkinitcpio.conf revert failed, **SKIP mkinitcpio -P entirely** and return
   `EXIT_BOOT_CRIT` (4). Confirm the taint flag is set on any prior boot-critical write
   failure so a half-written cmdline/mkinitcpio never reaches `mkinitcpio -P`.
2. `mkinitcpio -P` failure → abort remaining steps, `EXIT_BOOT_CRIT`, skip post-mki.
3. **`$BOOT` resolution refusal:** if `SDBOOT_REMOVE_EXISTING=yes` AND `_resolve_boot_path`
   returns empty (bootctl + findmnt both failed AND `/boot` missing), **refuse the
   boot-wipe gate** — return `EXIT_BOOT_CRIT` rather than run `sdboot-manage` with an
   unresolved `$BOOT`. This is the guard that prevents wiping the wrong/no target.
4. Post-rebuild sanity (`_preflight_boot_sanity`): vmlinuz + initramfs + entries must
   exist or `EXIT_BOOT_CRIT`.

Confirm: (a) there is NO path where `sdboot-manage` with `REMOVE_EXISTING=yes` runs
against an unverified `$BOOT`; (b) a mkinitcpio failure cannot leave the system with a
new cmdline but a stale/absent initramfs (ordering: is cmdline written in Phase 3
*before* the Phase 5 initramfs rebuild — meaning a Phase-5 failure leaves new cmdline +
old initramfs? trace this exact window and confirm the fallback entry §B1 covers it);
(c) `EXIT_BOOT_CRIT` aborts ALL remaining phases (no Finalize after a boot failure).

**mkinitcpio rollback (`_ip_snapshot_mkinitcpio` / `_mkinitcpio_revert`):** before
`pacman -Syu`, mkinitcpio.conf is snapshotted to `/run/ry-install` (0700 root:root,
mktemp). On pacman failure, `_mkinitcpio_revert` restores it via same-`/etc`-FS mktemp +
`_mr_copy_cmp_verify` (copy+compare) + `_mr_chmod_chown_mv` (atomic mv with
`--reference=destination` perms). Confirm the snapshot→revert path is complete: snapshot
in `/run` (tmpfs, lost on reboot — acceptable since revert is same-boot), revert tmp in
`/etc` (same-FS atomic), symlink-checked. Flag if a pacman partial-transaction could
desync mkinitcpio.conf from installed modules without triggering revert.

Actionable: trace the cmdline-write (Phase 3) vs initramfs-rebuild (Phase 5) ordering
window; confirm the `$BOOT`-unresolved refusal is the only path to `sdboot-manage
REMOVE_EXISTING`; confirm `EXIT_BOOT_CRIT` is terminal.

## K. Signal & exit teardown (`_cleanup`, `_teardown`, `_do_cleanup`)

- **Signal handlers:** `_cleanup` traps INT/TERM/HUP/QUIT/ABRT with correct **128+N exit
  codes** (129/130/131/143/134), idempotent via `_CLEANUP_DONE`. SIGPIPE
  (`_cleanup_pipe`) marks output broken and continues with JSONL-only. `fish_exit`
  (`_cleanup_on_exit`) ensures teardown on normal exit, preferring
  `_INTENDED_EXIT_CODE` → `_RY_INSTALL_LAST_EXIT` → `$status`.
- **Cleanup order (`_do_cleanup`):** kill children → mkinitcpio revert → tmpfile sweep →
  filesystem sweep → **lock release (sweeps run while lock still held)** → erase globals.
  Confirm: (a) children are reaped BEFORE revert so revert never races a live pacman;
  (b) the lock is released LAST so no second instance starts mid-sweep; (c) the tmpdir
  sweep globs (`_RY_TMPDIR_GLOBS`, 6 patterns) match every tmpfile the script creates
  (ry-sudo-err, ry-tee-err, ry-run, ry-argparse-err, ry-fstab-tee-err, ry-fstab-awk-err)
  — a missed glob leaks tmpfiles; an over-broad glob could delete another instance's
  tmp. Verify the glob set is exactly the created set.

Actionable: confirm SIGKILL (uncatchable) is the only signal that bypasses cleanup and
that the lock's stale-reclaim (§H) recovers from a SIGKILLed holder; confirm the tmpdir
glob set exactly matches the created-tmpfile set (no leak, no cross-instance deletion).

## L. pacman transaction safety (`_ip_pacman_invoke`)

- **Full `-Syu --needed` only (no partial upgrades):** first pass `-Syu`, retry `-Syyu`
  (forced db re-sync for mirror staleness). `--noconfirm` (unattended). Confirm the
  retry only addresses transient mirror staleness and never masks a real conflict (the
  warning says so — verify the second failure is fatal and surfaced).
- **db.lck pre-check:** refuses to run if `/var/lib/pacman/db.lck` exists (stale lock or
  live pacman). Confirm this is checked before any package op and that the script never
  removes db.lck itself (correct — it instructs the user to).
- **Arch partial-upgrade policy:** confirm full `-Syu` is mandatory on CachyOS/Arch and
  the script never issues `-S <pkg>` without `-yu` (which would risk a partial upgrade).
  PKGS_DEL removal (`_install_configure_services`) — confirm it uses `-Rns` or
  equivalent and does not trigger a partial-upgrade state.

Actionable: confirm retry-then-fatal semantics; confirm no self-removal of db.lck;
confirm PKGS_DEL removal cannot induce a partial-upgrade.

## Robustness verdict (required, in addition to the per-section §1–§12 verdicts)

Add a final **ROBUSTNESS** verdict block separate from the tuning verdicts:
- For each of §G–§L: PASS (guarantee holds) / GAP (window or fail-open found) /
  UNCERTAIN (cannot confirm against current fish/kernel without testing).
- Any GAP in §G/§H/§J (atomic-write, lock, boot-wipe) is release-blocking and outranks
  every tuning finding — surface it first regardless of IMPACT/RISK on config items.
- This layer is correctness, not preference: there is no "deliberate trade-off" defense
  for a partial-write window or a fail-open lock. FIX applies normally here (the
  flag-don't-FIX discipline is for config values, not safety invariants).

Sources for §G–§L: man7.org (mkdir(2), rename(2)/mv -T atomicity, proc(5) stat fields,
sysconf CLK_TCK), fishshell.com/docs (`$pipestatus`, `string collect`, `--on-signal`,
`--on-event fish_exit`), wiki.archlinux.org (pacman partial-upgrade policy, mkinitcpio,
systemd-boot), docs.kernel.org (/proc/stat btime, USER_HZ).
