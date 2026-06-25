ry-install changelog

All notable changes, newest first. Versions follow MAJOR.MINOR.PATCH.

7.70.1 - 2026-06-24

- fix: guard _err VERIFY_FAIL increment with set -q.

7.70.0 - 2026-06-23

- regdom: remove /etc/conf.d/wireless-regdom; /etc/iw-regdomain retained.
  SYSTEM_DESTINATIONS 15 -> 14, managed 18 -> 17.
- bluetooth: ReconnectAttempts 7 -> 3; remove ReconnectIntervals (BlueZ default backoff).
- preflight: raise mesa soft-floor warn 25.3 -> 26.0; non-fatal.

7.69.1 - 2026-06-22

- refactor: extract _content_fn_for helper to single-source the generator-name
  derivation shared by _ry_get_file_content and _ry_validate_configs. Output
  byte-identical across all 18 configs.

7.69.0 - 2026-06-22

- cmdline: drop amd_iommu=on (redundant on AMD; IOMMU on by default, iommu=pt
  retained for passthrough). KERNEL_PARAMS 13 -> 12.

7.68.0 - 2026-06-22

- cmdline: drop clearcpuid=rdseed. KERNEL_PARAMS 14 -> 13.
- preflight: drop _ry_check_rdseed_workaround_stale.

7.67.0 - 2026-06-22

- style: normalize RDSEED-microcode probe to fish (cmd) form; byte-identical.

7.66.0 - 2026-06-22

- verify: split iwd-process check into _vrsv_wifi_iwd_proc; behavior unchanged.

7.65.0 - 2026-06-21

- mangohud: order fps/frametime ahead of GPU/CPU block.
- guards: fix post-hook banner count (19 -> 18).

7.64.0 - 2026-06-21

- drirc: remove 95-ry-radv-apu.conf (gfx1151 reports uma:1 natively).
- network: remove dormant iwd/main.conf; NM_WIFI_BACKEND=iwd opt-in retained.
- guards: SYSTEM_DESTINATIONS 17 -> 15, _RY_POST_HOOKS 20 -> 18, managed 20 -> 18.

7.63.0 - 2026-06-21

- bluetooth: add main.conf (AutoEnable, FastConnectable, reconnect backoff).
- services: enable bluetooth.service. EXPECTED_SERVICES 4 -> 5.
- verify: add _vss_bluetooth.

7.62.0 - 2026-06-21

- cmdline: amd_iommu=off -> amd_iommu=on iommu=pt. KERNEL_PARAMS 13 -> 14.
- network: NM backend iwd -> wpa_supplicant; power-save off via wifi.powersave=2.
- services: mask modemmanager.service. MASK 9 -> 10.

7.61.0 - 2026-06-21

- systemd: add NetworkManager-dispatcher logging.conf (LogLevelMax=notice).
- fix: guard vercmp behind command -q in mesa soft-floor check.

7.60.0 - 2026-06-21

- mangohud: remove fps_metrics, cpu_temp, gpu_power, text_outline, toggle_hud.

7.59.0 - 2026-06-21

- cmdline: add clearcpuid=514 (UMIP off). KERNEL_PARAMS 12 -> 13.
- preflight: add _ry_check_umip_disabled (INFO while 514 set).
- sysctl: add vm.swappiness=150, vm.vfs_cache_pressure=50, vm.page-cluster=0.
  SYSCTL_VALUES 8 -> 11.
- gpu: remove ry-amdgpu-strixhalo.conf (kernel >= 6.16.9 auto-sizes GTT).
  SYSTEM_DESTINATIONS 16 -> 15.

7.58.0 - 2026-06-20

- refactor: move _configure_services_iwd_handoff to its Phase 4 slot.

7.57.0 - 2026-06-20

- pkg: add rtkit to PKGS_ADD. PKGS_ADD 16 -> 17.

7.56.0 - 2026-06-20

- cpu: governor performance -> powersave so EPP is honored under amd_pstate=active.
- cpu: EPP performance -> balance_performance (udev rule + verify).
- preflight: soft-warn mesa < 25.3 for gfx1151 RADV stability; non-fatal.

7.55.0 - 2026-06-20

- udev: scope GPU clock-floor rule to card device (KERNEL card[0-9], ACTION add).
- verify: _vss_udev asserts GPU rule card-scoped.

7.54.2 - 2026-06-17

- firewall: flush ufw only after nftables default-deny live; else retain + warn.
- install-file: boot/cmdline post-hook exiting boot-critical prints DO-NOT-REBOOT.
- udev: tighten NVMe match to nvme[0-9]*n[0-9]*.
- verify: _vre_fstab fails noatime+relatime/atime/strictatime.

7.53.0 - 2026-06-17

- nftables: scope inbound IPv4 ICMP to diagnostics; drop echo-request.
- preflight: amdgpu hard-fails when modinfo misses it; ICMP fallback 1.1.1.1 + 8.8.8.8.
- verify: drop systemd-analyze boot-time, THP/KSM checks.

7.52.0 - 2026-06-17

- ttm: relabel 32 GiB GTT value as a cap.
- docs: record Known Issues (MES, RTL8127, MT7925, ACP).

7.51.0 - 2026-06-16

- udev: EPP rule add -> add|change (re-asserts after AC/DC).
- verify: _vss_udev asserts GPU clock-floor; _vss_regdom asserts wireless-regdom.
- init: _ir_validate_post_hooks refuses deploy on hook tag with no handler.

7.48.0 and earlier - history trimmed

- See git tags / history for the full record.
