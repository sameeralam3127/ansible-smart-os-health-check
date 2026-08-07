# Changelog

All notable changes to the `sameeralam3127.linux_vitals` collection are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.1.0] - 2026-08-07

Distro-specific hardening of reboot-required, latest-kernel, and bootloader
default detection ([#2](https://github.com/sameeralam3127/linux-vitals/issues/2)).

### Added

- **Reboot detection now reports *how* it decided.** Every host carries
  `reboot_required_source`, `reboot_detection_supported`,
  `reboot_required_packages`, and a human-readable `reboot_reason` alongside
  `reboot_required`. These surface in the JSON report (`reboot.source`,
  `reboot.detection_supported`, `reboot.pending_packages`, `reboot.reason`),
  the generic webhook payload, the Slack host breakdown, and the dashboard's
  host detail.
- **Debian/Ubuntu pending packages.** The packages that requested the reboot
  are read from `/run/reboot-required.pkgs` and reported per host.
- **SUSE `zypper needs-rebooting`.** Used when available (exit `102` means a
  reboot is needed), falling back to the marker files for older zypper.
- **dnf5 hosts.** When the standalone `needs-restarting` binary is absent,
  `dnf needs-restarting --reboothint` is used before giving up.
- **BLS boot entries.** `saved_entry`/`GRUB_DEFAULT` values that name a Boot
  Loader Specification drop-in (RHEL 8+/Fedora, SUSE) now resolve by entry id
  and by entry title, not just by `grub.cfg` menuentry title.
- **systemd-boot.** `loader.conf`'s `default` -- including a glob such as
  `fedora-*`, resolved to the version-highest match -- is read from
  `loader/entries/`, so bootloader validation works on hosts with no GRUB.
- **[docs/kernel-reboot-detection.md](docs/kernel-reboot-detection.md)**,
  documenting the per-distro detection order and the known edge cases
  (live-patched kernels, transactional/immutable hosts, auto-discovered UKIs,
  `grub.cfg` submenus, containers, and more).
- Tests covering the derived reboot facts, each distro probe, and bootloader
  resolution (BLS id, `grub.cfg` index, systemd-boot glob, no bootloader),
  plus a POSIX-syntax check of every embedded discovery script.

### Fixed

- **An unusable distro check was read as "no reboot needed".** A missing
  `needs-restarting` (RHEL minimal images without `dnf-utils`) or a
  non-`0`/`1` exit produced `reboot_required: false` -- indistinguishable from
  a genuinely up-to-date host. Those hosts now fall back to the running-vs-latest
  kernel comparison and report `reboot_required_source: kernel-comparison`.
- **Stale `/lib/modules` directories could masquerade as the latest kernel.** A
  directory left behind by an interrupted removal (no `modules.dep`, no image in
  `/boot`) made a fully patched host report its kernel as outdated forever.
  Directories are now filtered to ones that still look like an installed kernel,
  with the unfiltered listing kept as a fallback.
- **Kernel-comparison fallback referenced a fact defined in the same
  `set_fact`.** The non-Debian/RedHat/SUSE branch of `linux_vitals_reboot_required`
  read `linux_vitals_latest_kernel_selected` from the task that was defining it
  (the same class of bug as 1.0.1/1.0.2). Reboot facts are now derived in their
  own task file after discovery facts are set.
- **`--tags kernel` skipped the bootloader finalize tasks.** The two
  `Finalize bootloader ...` tasks carried no tags, so the documented
  `--tags kernel,reporting` run left `linux_vitals_bootloader_default_matches_latest`
  undefined and failed while building the result. Both are now tagged
  `discovery`/`kernel`.
- **Bootloader kernel paths weren't normalised.** Entries pointing at a symlink
  (Debian's `/vmlinuz`), at a path relative to a separate `/boot` partition, or
  at a `.efi`/`kernel-`-prefixed image compared unequal to the installed kernel
  version and produced spurious "default boot entry does not select the latest
  installed kernel" findings.

### Changed

- JSON report schema is now `1.2` (additive: the new `reboot.*` fields).

## [1.0.2] - 2026-07-12

### Fixed

- **`.env`-based notification config never actually worked.** `linux_vitals_dotenv_slack_webhook_url` (and the four other `linux_vitals_dotenv_*` secrets: generic webhook URL, SMTP host/username/password) referenced `linux_vitals_dotenv_contents` from *within the same `set_fact` call* that defined it -- the same same-task self-reference bug as the 1.0.1 bootloader fix, except here it failed silently (Ansible's `regex_findall` on an Undefined value fell through the `| default('', true)` chain to an empty string) instead of erroring, so it went undetected through every prior test and release. Every channel that relied on `.env` as a fallback (rather than an explicit inventory/`group_vars` value) was silently skipped. Split `vitals_report/tasks/config.yml`'s first task into two sequential tasks. Verified by re-running against the live Multipass fleet: the Slack notification task changed from `skipping` to `ok` and the webhook received an actual HTTP POST.

## [1.0.1] - 2026-07-12

Fixes found by running `playbooks/baseline.yml` / `playbooks/postcheck.yml` against a real 10-node Ubuntu fleet (Multipass) for the first time -- none of these were reachable from the macOS-only testing available during initial development.

### Fixed

- **Service-name resolution was non-deterministic.** Every `<candidates> | intersect(<present services>) | first` lookup (time-sync, `sssd`, `systemd-journald`) used `intersect()`, whose result order is not guaranteed to follow either input list -- it could return a different match on every run against the identical host state, occasionally picking a present-but-inactive unit (e.g. `ntp.service`, stopped) over the actually-running one. Replaced with `<candidates> | select('in', <present services>) | list | first`, which deterministically preserves the candidate list's priority order.
- **`linux_vitals_time_sync_candidates` never included Debian/Ubuntu's actual service names.** It only listed `chronyd*` (RHEL's name) and `ntp*`, missing `chrony.service`/`chrony` (the real unit installed by Debian/Ubuntu's `chrony` package) and `systemd-timesyncd.service`/`systemd-timesyncd` (Ubuntu's out-of-the-box default). Every stock Ubuntu host was reporting a false "chronyd or ntp is not installed" finding despite `systemd-timesyncd` running the whole time. Both are now in the candidate list.
- **Systemd alias units report `state: active`, not `state: running`.** `chronyd.service` is often a symlink alias to `chrony.service` on Debian/Ubuntu; `ansible.builtin.service_facts` reports alias units with `state: active` rather than `running`. The `sssd`/`systemd-journald`/time-sync "is it up" checks, and the self-healing restart-success check, only accepted `running` and treated a perfectly healthy aliased service as down. Both now accept `state in ['running', 'active']`.
- **Same-task variable self-reference in bootloader validation.** `linux_vitals_bootloader_validation_message` referenced `linux_vitals_bootloader_default_matches_latest` from within the *same* `set_fact` call that defined it -- Ansible templates every key in a `set_fact` dict against the variable context that existed *before* the task ran, so this was always undefined at evaluation time. It only surfaced as a hard failure once bootloader detection actually reached `status: resolved` (i.e. on a real host with working `grubby`/`grub2-editenv`, never on the macOS dev machine). Split into two sequential tasks.

## [1.0.0] - 2026-07-12

Published to [Ansible Galaxy](https://galaxy.ansible.com/ui/repo/published/sameeralam3127/linux_vitals/).

### Added

- Initial release of LinuxVitals as an Ansible collection (`sameeralam3127.linux_vitals`), rebranded from the `smart_os_health_check` role-based project.
- Three composable roles: `vitals_scan` (read-only discovery and findings), `vitals_heal` (opt-in one-shot self-healing, disabled by default), and `vitals_report` (HTML/JSON dashboard, archive retention, Slack/email/generic-webhook notifications).
- `playbooks/healthcheck.yml`, runnable locally or via FQCN once installed (`sameeralam3127.linux_vitals.healthcheck`).
- Example inventory and `group_vars` under `examples/`.
- Baseline/postcheck maintenance workflow: `playbooks/baseline.yml` and `playbooks/postcheck.yml` snapshot each host's result under a shared `linux_vitals_maintenance_id` and automatically compute a before/after comparison (status change, RAM delta, kernel change, reboot-required change, new/resolved findings) for the postcheck run.
- Redesigned, self-contained HTML dashboard: executive KPI row with a health-score ring (pass-rate band: Excellent/Good/Fair/Poor), a searchable and sortable host table with expandable per-host detail rows, status/comparison filter chips, light/dark themes (OS-aware plus a manual toggle), a print/export stylesheet, and a per-host serial number field as a first piece of asset-level drilldown data.
- JSON report (schema 1.1) now includes maintenance phase/id, health-score and comparison rollups, and each host's `asset_serial` and `comparison` object.
- Full documentation suite under `docs/` (installation, quickstart, configuration reference, variable reference, report guide, examples, troubleshooting, architecture) plus `CONTRIBUTING.md`.
- `examples/playbooks/custom-thresholds.yml`, a working example of overriding thresholds and enabling self-healing for a single run.
- Per-role `README.md` for `vitals_scan`, `vitals_heal`, and `vitals_report` (required by Ansible Galaxy's import validation -- the first publish attempt failed with "No role readme found").

### Packaging

- Excluded `tests/`, `pytest.ini`, and `requirements.yml` from the published tarball via `galaxy.yml`'s `build_ignore` -- they're dev/CI-only; `galaxy.yml`'s own `dependencies:` block is what Galaxy uses to resolve `community.general` for installed-collection consumers.
- Verified end-to-end: built the tarball with `ansible-galaxy collection build`, installed it into an isolated collections path, and confirmed all three playbooks resolve and execute via FQCN (`sameeralam3127.linux_vitals.healthcheck` / `.baseline` / `.postcheck`) from that installed artifact, independent of the dev symlink.

### Changed

- All `smart_os_health_check_*` variables, facts, templates, and default report filenames renamed to the `linux_vitals_*` / `linux_vitals_report.*` convention.
- Report output path and the `.env` secrets loader now resolve relative to `inventory_dir` instead of `playbook_dir`, so both local development and installed-collection usage read/write files in the operator's own project rather than inside the installed package.

### Fixed

- Removed a byte-identical duplicate bootloader-detection task that ran the grub probe twice per host.
- Replaced a gawk-only bootloader parser with a portable POSIX `sh` implementation (bash/dash/system `sh` verified) and added numeric `GRUB_DEFAULT` index resolution.
- Stopped counting `lastb`'s trailing `btmp begins ...` footer line as a failed login attempt.
- Fixed a Jinja filter-precedence bug where `* 100 | round(1)` rounded the literal `100` instead of the computed percentage.
- HTML-escaped journal/`lastb`-derived free text in the dashboard template.
- Added `no_log: true` to the tasks that load webhook/SMTP secrets from `.env`.

### Security

- Removed real lab IP addresses previously committed in `inventory/hosts.ini` and `inventory/multipass-10.ini`; replaced with RFC 5737 documentation-range examples under `examples/inventory/`.
