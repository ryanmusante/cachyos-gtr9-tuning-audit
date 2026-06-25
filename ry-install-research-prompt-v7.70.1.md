# Deep-Research Prompt — ry-install Tuning (CachyOS · Beelink GTR9 Pro)

Target: `ry-install.fish` **v7.70.1** (attached). Source of truth: script > README > CHANGELOG.

**Platform:** Beelink GTR9 Pro · Ryzen AI Max+ 395 "Strix Halo" (Zen 5, 16C/32T, gfx1151) · Radeon 8060S (40 RDNA 3.5 CUs) · XDNA 2 NPU · 128 GB LPDDR5X-8000 unified (≤96 GB as VRAM) · dual M.2 NVMe (ext4) · dual 10 GbE (RTL8127) + Wi-Fi 7 (MT7925) + BT 5.4 · 140 W TDP · CachyOS · systemd-boot.

**Counts (from `_ir_validate_counts`):** KERNEL_PARAMS 12 · ENV_VARS 10 · SYSCTL 11 · PKGS_ADD 17 · PKGS_DEL 9 · MASK 10 · EXPECTED_SERVICES 5 · managed files 17 (14 system + 3 user) · MangoHud 15 directives · MKINITCPIO_HOOKS 11 · LOGIND 8.

## Mission

Evaluate every config decision against current upstream sources for this exact silicon. Return a prioritized, evidence-backed tuning report. The profile deliberately trades power-saving and host firewalling for performance/latency — confirm each choice is current/correct, surface anything superseded/harmful, quantify safety deltas; do not second-guess intentional design.

## Rules

1. Item-by-item, hardware-anchored to gfx1151 / Zen 5 / RDNA 3.5 / CachyOS / 128 GB unified / dual 10 GbE.
2. Respect deliberate trade-offs: **flag + quantify**, do not auto-FIX. Reserve FIX for incorrect/superseded/deprecated/harmful values.
3. Rate IMPACT × RISK (High/Med/Low). Default KEEP when impact marginal and risk non-trivial.
4. Never invent params/flags/keys/options/URLs — cite a source or mark UNCERTAIN.
5. Flag every source conflict and state which you trust.
6. Exact versions (kernel / Mesa / linux-firmware / pkg) and exact before→after, mapped to the in-script global.

## Output

- **Findings matrix** (box-drawn Unicode, code fence, grouped by section): ITEM · CURRENT (v7.70.1 value) · CALL (KEEP/TUNE/FIX/UNCERTAIN) · RECOMMENDED · IMPACT · RISK · EVIDENCE (URL + version/date/commit).
- **Before→after** for each TUNE/FIX: exact current string, exact replacement, in-script global.
- **VERIFY block** (post-reboot commands, below).
- **Security delta vs CachyOS defaults** (ordered list, below).
- **Verdict**: one per section (OPTIMAL/TUNE/FIX) + overall (PASS/PASS-WITH-FIXES/FAIL).
- **Methodology**: source list with access dates/versions; conflicts flagged; unknowns marked UNCERTAIN.

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
cat /sys/class/drm/card*/device/power_dpm_force_performance_level   # high
cat /proc/cmdline | rg -o 'iommu=\S+'                       # iommu=pt
cat /proc/cmdline | rg -o 'clearcpuid=\S+'                  # clearcpuid=514 (UMIP off)
ls -l /dev/ntsync                                           # present (observed, not deployed)
vulkaninfo | rg -i 'driverName|deviceName'                 # RADV / Radeon 8060S; confirm uma heap
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc vm.max_map_count vm.compaction_proactiveness vm.swappiness vm.page-cluster vm.vfs_cache_pressure
findmnt -no OPTIONS /                                       # noatime,lazytime,commit=10
iw reg get | rg -i country                                 # US
cat /etc/iw-regdomain                                      # COUNTRY=US (single regdom file; conf.d/wireless-regdom removed)
sudo nft list chain inet filter input                      # policy drop + ICMPv6 NDP/PMTUD + IPv4 diag-only (no inbound ping)
systemctl is-enabled bluetooth.service                     # enabled
printenv MANGOHUD                                          # 1
```

Hard `--verify` asserts (mismatch → exit 1): scaling_driver, scaling_governor=powersave, EPP=balance_performance, amd_pstate status/prefcore, boost, clocksource, GPU power_dpm=high, regdom (`/etc/iw-regdomain`; the second regdom file is removed — do not verify `conf.d/wireless-regdom`), nftables `nd-neighbor-solicit` present. Warn-level only: ZRAM, live nftables NDP cross-check, iwd runtime state. No boot-time/THP/KSM checks (removed). No `ttm.*`/drirc/`radv_enable_unified_heap_on_apu` globals exist (removed) — do not verify them.

### Security delta (ordered)

1. **UMIP off** (`clearcpuid=514`) — descriptor-table base leak, kernel tainted; headline open reduction.
2. **split_lock_detect=off** — a misbehaving app can degrade the system.
3. **Plaintext DNS** (`DNSOverTLS=no`, `DNSSEC=allow-downgrade`) reverting CachyOS DoH default — DNS observable/spoofable on-path.
4. **Firewall tightened** (nftables default-deny-inbound ships; no inbound IPv4 ping) — net positive.
5. **IOMMU restored** (`iommu=pt`, AMD-Vi on) — the prior `amd_iommu=off` reduction is **closed**.

## Investigation (ordered by installer phase)

### 1. Platform baseline & version floors
Current: no hard kernel floor; CPU gate `Ryzen AI Max`; soft mesa<26.0 warn (preflight); MES 0x86 fix via linux-firmware + mkinitcpio-firmware.
- Establish the min mainline + CachyOS kernel that fully enables gfx1151 display/amdgpu/amd_pstate-EPP/ACP; check the installed kernel; assess whether a hard floor should be reinstated (the r8169 landing version, §11, is the strongest concrete floor).
- Confirm the soft mesa 26.0 floor matches current RADV guidance (the floor was raised 25.3 → 26.0 in v7.70.0); enumerate open gfx1151 RADV issues and state whether 26.0 is the right threshold or now too low/high.
- Confirm gfx1151 reports `uma:1` natively so the removed drirc override (§5) is genuinely redundant on current Mesa.
- Give the exact linux-firmware version first carrying the MES 0x86 blob; check the installed version.
- Sources: wiki.cachyos.org, docs.kernel.org gpu/amdgpu, gitlab.freedesktop.org/mesa (gfx1151), git.kernel.org linux-firmware.

### 2. Packages
Current PKGS_ADD (17): nvme-cli, cachyos-gaming-meta, cachyos-gaming-applications, lib32-mesa, mkinitcpio-firmware, fd, sd, dust, procs, bottom, htop, git-delta, lm_sensors, rtkit, realtime-privileges, ddcutil, nftables. PKGS_DEL (9): plymouth stack + micro + cachyos-micro-settings + cachy-update + kdeconnect. AUR none. Vulkan (chwd): vulkan-radeon, lib32-vulkan-radeon.
- Confirm cachyos-gaming-meta/-applications supply RADV/Proton/gamescope/MangoHud/GameMode and that the meta pulls MangoHud (profile also ships MangoHud.conf — confirm no conflict).
- `rtkit` (new): confirm RealtimeKit is correct alongside realtime-privileges for PipeWire thread priority; does `rtkit-daemon.service` need enabling, or is it socket-activated?
- CachyOS repo tier: does `x86-64-v4`/znver (AVX-512) apply and benefit this build vs v3?
- Confirm `lib32-mesa` still needed alongside `lib32-vulkan-radeon` or now redundant.
- Confirm plymouth*/micro/cachy-update removal has no dep fallout.
- Sources: wiki.cachyos.org, wiki.archlinux.org/Gaming + PipeWire + RealtimeKit.

### 3. Kernel cmdline (12)
`8250.nr_uarts=0 amd_pstate=active iommu=pt clearcpuid=514 nowatchdog nvme_core.default_ps_max_latency_us=0 pcie_aspm.policy=performance quiet split_lock_detect=off tsc=reliable usbcore.autosuspend=-1 zswap.enabled=0`
- **iommu=pt**: confirm AMD-Vi is left enabled by default (so dropping explicit `amd_iommu=on` is genuinely redundant, not a silent revert to off); quantify gaming/compute cost of `pt` vs no param; confirm it's the recommended posture for ROCm + USB4 isolation.
- **clearcpuid=514 (UMIP off)**: establish the real benefit (avoiding umip_printk stutter under anti-cheat/Wine) vs the security cost (descriptor-table leak, kernel taint). Confirm whether current Proton/EAC/BattlEye actually trip UMIP emulation on Zen 5. Flag; recommend dropping if no stutter observed.
- `amd_pstate=active`: confirm recommended on Zen 5 and its interaction with powersave governor + balance_performance EPP (§6).
- `split_lock_detect=off`: perf vs stability; current default; blast radius.
- No `preempt=` pinned: confirm leaving preemption to CachyOS BORE/EEVDF costs nothing.
- Zero amdgpu/ttm module params: confirm hands-off GPU-param posture is correct (GTT via kernel default §5, GPU clock via udev §6).
- Validate the rest as current: tsc=reliable, nowatchdog, 8250.nr_uarts=0, usbcore.autosuspend=-1, nvme_core.default_ps_max_latency_us=0, pcie_aspm.policy=performance, zswap.enabled=0.
- Sources: docs.kernel.org kernel-parameters + pm/amd-pstate + x86 UMIP, wiki.archlinux.org/AMDGPU + IOMMU.

### 4. Bootloader & initramfs
loader.conf: default @saved, timeout 0, console-mode keep, editor no. sdboot-manage: DEFAULT_ENTRY manual, OVERWRITE/REMOVE_EXISTING/REMOVE_OBSOLETE yes, LINUX_FALLBACK_OPTIONS "quiet". mkinitcpio: MODULES=(amdgpu), HOOKS (11) = base systemd autodetect microcode modconf kms keyboard sd-vconsole block filesystems fsck, COMPRESSION zstd -1 -T0.
- Verify HOOKS order with the systemd hook (microcode/kms/sd-vconsole/block placement); confirm amdgpu+kms is the recommended early-KMS for this GPU.
- zstd -1 -T0: confirm optimal level for NVMe; quantify any higher-level boot-time win (no boot-time target is pinned).
- timeout 0 + DEFAULT_ENTRY manual + REMOVE_EXISTING=yes wipes foreign BLS entries (EFI loaders untouched) — confirm current/intended.
- Confirm sdboot-manage is current/maintained vs kernel-install/UKI (UKI out of scope).
- Sources: wiki.archlinux.org/Mkinitcpio + systemd-boot, sdboot-manage upstream.

### 5. GPU / Vulkan / gaming
No drirc shipped (95-ry-radv-apu.conf removed — gfx1151 reports uma:1 natively). No ttm/modprobe params (removed — kernel ≥6.16.9 auto-sizes GTT to ~50%-of-RAM, ~62 GiB). ENV_VARS (10): AMD_VULKAN_ICD=RADV, MANGOHUD=1, DXVK_LOG_LEVEL=none, DXVK_LOG_PATH=none, MESA_SHADER_CACHE_MAX_SIZE=16G, PROTON_ENABLE_WAYLAND=1, PROTON_LOCAL_SHADER_CACHE=1, VKD3D_DEBUG=none, VKD3D_SHADER_DEBUG=none, WINEDEBUG=-all.
- **RADV unified heap (drirc removed)**: confirm current RADV on gfx1151 reports uma:1 and treats heap as unified *without* the override (via vulkaninfo heap flags / Mesa source). If not, flag a regression.
- **GTT sizing (ttm removed)**: confirm the installed kernel auto-sizes GTT sensibly; establish the actual ceiling (~62 GiB). Is that right for large-VRAM gaming AND for ROCm/llama.cpp wanting ~96 GiB, or does removing the cap under-provision compute? If a workload needs >~62 GiB, determine the current correct mechanism (BIOS UMA carveout vs ttm.* vs sysfs) and whether the profile should reintroduce an opt-in knob. Confirm amdgpu.gttsize still deprecated.
- **ntsync** (observed, not deployed): confirm it's the current Wine-sync mechanism (vs esync/fsync), which kernel mainlined it / which CachyOS kernel ships it, that Proton consumes /dev/ntsync and improves frametimes on 16C/32T; assess whether the profile should take a position (autoload conf / verified env var) or leave to CachyOS defaults. Largest unmanaged gaming surface — flag.
- **PROTON_FSR4_RDNA3_UPGRADE removed**: confirm current Proton/Proton-CachyOS FSR4-upgrade support for RDNA 3.5 — is there now a verified env var? If yes, recommend (flag); if still unverified, removal stands.
- PROTON_ENABLE_WAYLAND=1: maturity + fallback on current Proton.
- AMD_VULKAN_ICD=RADV: confirm it reliably forces RADV vs VK_DRIVER_FILES.
- MESA_SHADER_CACHE_MAX_SIZE=16G + PROTON_LOCAL_SHADER_CACHE=1: confirm no conflict, sane sizing.
- MANGOHUD=1 global enable: confirm low-overhead vs per-launch, clean with gamescope/GameMode.
- Sources: docs.mesa3d.org (RADV, APU heap), gitlab.freedesktop.org/mesa + drm/amd (GTT auto-sizing kernel version), github Proton/Proton-CachyOS (FSR4, ntsync), amd.com ROCm, wiki.cachyos.org.

### 6. CPU performance & power
amd_pstate=active; governor **powersave** (honors EPP under active mode); EPP **balance_performance** via udev (add|change, re-asserts after AC/DC); GPU clock-floor power_dpm_force_performance_level=high (udev, card[0-9], add-only). Masked: power-profiles-daemon, ananicy-cpp, modemmanager. Installed: realtime-privileges, rtkit.
- **Note**: governor=powersave + EPP=balance_performance is the EPP-honoring config under active mode (the performance governor pins max and ignores EPP). Do NOT flag powersave as a power regression or recommend reverting to performance without establishing it wouldn't override the EPP hint.
- Confirm the active + powersave + balance_performance triple is the documented EPP-honoring max-perf config on Zen 5. State whether balance_performance (mid-bias) leaves boost residency on the table vs EPP=performance, and whether mid-bias is the better thermal/latency trade on a 140 W mini-PC. **Central question.**
- Confirm udev add|change pinning is robust vs one-shot; prefcore=enabled + boost=1 correct on Strix Halo.
- Mask power-profiles-daemon: confirm no needed platform_profile path lost; does CachyOS expect ppd? Consider tuned.
- Mask ananicy-cpp: confirm net win for gaming.
- Mask modemmanager (new): confirm no cellular HW so masking is loss-free.
- GPU clock-floor=high: (a) does forcing high vs auto help frametime/1%-lows on gfx1151, or just raise idle power? (b) on a shared 140 W package does pinning GPU high steal CPU boost headroom, hurting CPU-bound titles — *especially with EPP now balance_performance*? (c) high vs manual+pp_dpm_sclk vs auto? Flag, quantify thermal cost (--verify treats ≠high as drift).
- Sources: docs.kernel.org pm/amd-pstate (active + EPP + governor interaction), wiki.archlinux.org/CPU_frequency_scaling, freedesktop ppd.

### 7. Memory & storage
zswap.enabled=0; NVMe scheduler none (udev); vm.compaction_proactiveness=0, vm.max_map_count=2147483642, vm.page-cluster=0, vm.swappiness=150, vm.vfs_cache_pressure=50 (zram-tuned, priority 95 after vendor 70-cachyos-settings.conf); fstab ext4 noatime,lazytime,commit=10; THP/KSM/systemd-oomd left to CachyOS (oomd intentionally NOT enabled — kernel OOM + zram).
- **New zram trio**: confirm running kernel accepts swappiness>100; is 150 gratuitous on 128 GB RAM or does it help large-alloc/LLM reclaim? Confirm page-cluster=0 (disable readahead) is the standard zram rec. Confirm vfs_cache_pressure=50 is a net win vs default 100 and doesn't starve page cache. List which vendor 70-cachyos values the priority-95 file overrides and whether justified.
- zswap.enabled=0: confirm CachyOS uses zram (not zswap) by default — no double-compression conflict.
- NVMe scheduler none: confirm best practice vs mq-deadline/kyber on this kernel; evaluate nr_requests/read_ahead_kb.
- fstab noatime,lazytime,commit=10: confirm noatime+lazytime coexist; weigh commit=10 durability; confirm fstrim.timer over continuous discard.
- vm.max_map_count near 2^31: confirm appropriate for Proton/anti-cheat, not gratuitous. compaction_proactiveness=0: confirm right for gaming + large unified allocs.
- Confirm not enabling systemd-oomd (kernel OOM + zram) is right on 128 GB.
- Sources: docs.kernel.org (block, sysctl/vm), wiki.archlinux.org/Zram + SSD + Ext4, wiki.cachyos.org (zram defaults).

### 8. Network & latency
sysctl net: default_qdisc=fq, netdev_budget=600, netdev_budget_usecs=5000, tcp_congestion_control=bbr, tcp_notsent_lowat=16384, tcp_slow_start_after_idle=0. NM: wifi.backend=wpa_supplicant (NM default; iwd opt-in via NM_WIFI_BACKEND), wifi.powersave=2 (off), logging WARN. resolved: MulticastDNS=no, LLMNR=no, DNSOverTLS=no, DNSSEC=allow-downgrade (plaintext; diverges from CachyOS DoH default). regdom: COUNTRY=US fixed (single file `/etc/iw-regdomain`; `conf.d/wireless-regdom` removed in v7.70.0). Masked: NetworkManager-wait-online, modemmanager. Enabled: NetworkManager.
- **NM backend wpa_supplicant** (reverted from iwd): verify current Arch/CachyOS guidance for MT7925/mt76 — is wpa_supplicant the more stable backend today, or has iwd matured to parity? (Profile has flip-flopped 3x — weigh evidence, not momentum.) Confirm no leftover iwd/NM race and that removed iwd/main.conf leaves no dangling ref. Confirm wifi.powersave=2 alone fully disables the mt76 software power-save latency issue under wpa_supplicant.
- bbr + fq: confirm still recommended; clarify BBRv3 status in current kernels.
- Dual 10 GbE: validate netdev_budget/usecs for 10G; assess tcp_rmem/wmem or NIC ring tuning for line-rate (RTL8127 issue §11).
- tcp_notsent_lowat=16384, tcp_slow_start_after_idle=0: confirm rationale holds.
- mDNS off (consistent with nftables drop §10); plaintext DNS reverting DoH — confirm CachyOS ships an encrypted-DNS default this overrides; flag privacy reduction (§10).
- regdom US fixed: confirm correct max TX power / channel set for MT7925 on current cfg80211/wireless-regdb (the "3 dBm" readout is cosmetic — don't attribute to regdom). Flag if 6 GHz Wi-Fi 7 needs a specific AFC posture beyond a country code; note non-US deployers must hand-edit COUNTRY.
- Sources: docs.kernel.org/networking (bbr, fq, tcp), wiki.archlinux.org/Sysctl + NetworkManager + Wireless/MediaTek, git.kernel.org wireless-regdb.

### 9. systemd units
Mask (10): ananicy-cpp, power-profiles-daemon, NetworkManager-wait-online, ufw, modemmanager, sleep/suspend/hibernate/hybrid-sleep/suspend-then-hibernate targets. Enable (5): fstrim.timer, NetworkManager, cpupower, nftables, bluetooth. Not enabled: systemd-oomd (intentional), NetworkManager-dispatcher (socket-activated), rtkit-daemon (assess §2). iwd.service untouched (opt-in). ufw flushed after nftables live.
- For each mask confirm safe/beneficial on CachyOS: ananicy-cpp + ppd (§6); modemmanager (no cellular HW); sleep/suspend masked = no suspend at all (matches always-on mini-PC, breaks nothing in logind).
- bluetooth.service now enabled (with main.conf §12): confirm AutoEnable=true posture and BlueZ key currency.
- Confirm nftables enabled as firewall and ufw-flush-then-mask leaves no unfirewalled window.
- fstrim.timer vs continuous discard; cpupower vs CachyOS freq mgmt; confirm oomd stays disabled and dispatcher stays socket-activated.
- logind Handle*Key=ignore incl LongPress: confirm intended, no lockout risk.
- Sources: man.archlinux.org (systemd.unit, logind.conf), wiki.archlinux.org (Bluetooth), wiki.cachyos.org.

### 10. Security & safety (cross-cutting)
nftables default-deny-inbound (ufw masked): input policy drop, ct established/related accept, lo accept, ct invalid drop, ICMPv6 NDP/PMTUD+echo-request accept, IPv4 ICMP diagnostics-only (no inbound ping), forward drop, output accept. iommu=pt (AMD-Vi on, DMA isolation restored). clearcpuid=514 (UMIP off). split_lock_detect=off.
- Validate nftables shape minimal-but-sufficient on a dual-10 GbE LAN; flag any port gaming/Proton/remote-play (Steam Remote Play, Sunshine/Moonlight) may need inbound.
- IPv4 ping dropped, IPv6 reachable: confirm intended asymmetric LAN posture.
- ICMPv6 NDP/PMTUD accept must be present (static _vss_nft hard-fails if nd-neighbor-solicit missing; live check warn-only). Treat static rule as the gate.
- iommu=pt now a net positive vs prior amd_iommu=off — confirm AMD-Vi genuinely enabled, record as closed reduction.
- clearcpuid=514 the new open reduction — quantify exposure vs umip_printk stutter prevented (§3); list first in security delta.
- split_lock_detect=off: quantify residual exposure.
- Produce the ordered security-delta subsection (above).
- Sources: wiki.archlinux.org (nftables, Security, IOMMU), docs.kernel.org (split lock, UMIP).

### 11. Known issues & DKMS currency
MES page faults → resolved (MES 0x86; linux-firmware + mkinitcpio-firmware). RTL8127 throughput → resolved in-tree r8169 (f24f7b2f3af9 + ae1737e7339b); no DKMS. MT7925 panics/deauth → open (out-of-tree DKMS; "3 dBm" readout cosmetic). Strix Halo ACP → open (HDMI/USB audio unaffected). Install pacman-only.
- Re-confirm resolved claims: MES 0x86 blob version carried by installed linux-firmware + mkinitcpio-firmware; the min kernel containing both r8169 commits (→ concrete kernel floor, §1). Give exact landing versions.
- MT7925 + ACP: check current upstream status (open/partial/landed). For MT7925 focus on panics/deauth (not the cosmetic readout); prefer a kernel/firmware floor over DKMS given pacman-only posture. Note the wpa_supplicant revert (§8) is itself an MT7925-stability move — assess whether backend + kernel floor close the gap.
- Recommend a kernel/firmware floor over DKMS for any landed fix; confirm any suggested DKMS still builds.
- Sources: gitlab.freedesktop.org/drm/amd, git.kernel.org linux-firmware + r8169 (f24f7b2f3af9, ae1737e7339b), bugzilla.kernel.org, discuss.cachyos.org.

### 12. MangoHud, Bluetooth & hygiene
MangoHud.conf (15 directives, 0600, fps/frametime first): horizontal, legacy_layout=0, position=top-left, fps, frametime, frame_timing, gpu_stats, gpu_core_clock, gpu_temp, cpu_stats, cpu_mhz, vram, ram, font_size=20, background_alpha=0.4. Removed since prior: gpu_power, cpu_temp, text_outline, toggle_hud, fps_metrics. Enabled via MANGOHUD=1. bluetooth main.conf: FastConnectable=true, AutoEnable=true, ReconnectAttempts=3 (no explicit ReconnectIntervals — BlueZ default backoff; v7.70.0 dropped the 1,2,4,8,16,32,64 list and lowered attempts 7 → 3). baloofilerc: Indexing-Enabled=false.
- Confirm each MangoHud directive valid for current MangoHud (don't invent). With gpu_power + cpu_temp dropped, assess whether their removal loses the two most decision-relevant readouts on a 140 W shared-package APU — possible TUNE-to-restore. Note toggle_hud removal means no Shift_R+F12 hide bind — assess whether a toggle should return. Establish the real min MangoHud + kernel for the remaining amdgpu sensors on gfx1151.
- Confirm gpu_temp/gpu_core_clock/vram/cpu_mhz populate from amdgpu sensors under Wayland on this iGPU; confirm vram+ram is the right unified-memory representation.
- Overhead: confirm near-zero, no conflict with gamescope overlay/GameMode.
- Bluetooth main.conf (new): confirm BlueZ keys/sections current; with ReconnectIntervals removed (v7.70.0), confirm BlueZ's built-in default reconnect backoff is sane for paired audio sinks and that ReconnectAttempts=3 (lowered from 7) is sufficient for reliable sink re-pair at boot without excessive retry; AutoEnable=true fixes adapter-off-at-boot; FastConnectable=true has no meaningful downside on an always-on desktop. Flag if dropping the explicit interval list materially changes reconnect timing on current BlueZ.
- baloofilerc Indexing-Enabled=false: confirm [Basic Settings]/Indexing-Enabled current for installed Baloo.
- **Naming**: every human-facing string reads "GTR9 Pro" but internal PROFILE_NAME + function names carry `gtr_pro` (FIX, Low/Low) — do not change if it breaks function-name refs or log fields; cosmetic.
- Sources: github flightlessmango/MangoHud (config ref, version tags), wiki.archlinux.org/MangoHud + Bluetooth, man.archlinux.org bluetooth main.conf, KDE Baloo docs.

## Scope & non-goals

- Recommendations only — do not emit a modified script.
- Out of scope: dotfiles, shells, editors, secrets, backups, multi-user, non-CachyOS, laptops, UKI.
- Per-game Proton tuning secondary; prioritize system-wide config.
- Do not recommend reinstating deliberately removed values (amdgpu.ppfeaturemask, PROTON_FSR4_RDNA3_UPGRADE, --country flag, TTM/GTT cap, RADV drirc, MangoHud gpu_power/cpu_temp/toggle_hud) unless current upstream directly contradicts the removal rationale — and then flag, don't FIX.
- IOMMU special case: amd_iommu=off is itself removed; current is iommu=pt — treat changes under the reinstatement rule, record as restored mitigation.
- Governor/EPP special case: performance→powersave + performance→balance_performance was deliberate (EPP-honoring under active mode) — do not flag powersave as a regression without proving the performance governor wouldn't override the EPP hint.
