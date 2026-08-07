# Kernel and Reboot Detection

How `vitals_scan` decides whether a host needs a reboot, which kernel is the
latest installed one, and whether the bootloader will actually boot it -- plus
the edge cases where each answer degrades.

All of the probes below are read-only, run with `changed_when: false` and
`failed_when: false`, and are written for the managed host's `/bin/sh` (dash
on Debian/Ubuntu), so a missing tool downgrades the answer instead of failing
the run.

## Reboot required

Each family gets its own probe. Every probe reports three things: whether the
check is *supported* on this host, whether a reboot was *detected*, and the
*source* that produced the answer. The source is carried through to
`reboot_required_source` in the JSON report, the webhook payload, the Slack
line, and the dashboard's host detail.

| Family | Primary check | Fallback |
| --- | --- | --- |
| Debian / Ubuntu | `/run/reboot-required` (then `/var/run/reboot-required`), with the packages from the matching `.pkgs` file | -- |
| RHEL / Fedora / CentOS | `needs-restarting -r` (`dnf-utils`/`yum-utils`) | `dnf needs-restarting --reboothint` on dnf5 hosts where the standalone binary is gone |
| SUSE / openSUSE | `zypper needs-rebooting` (exit `102` = reboot needed) | `/run/reboot-needed`, `/var/run/reboot-needed`, `/run/reboot-required`, `/var/run/reboot-required` |
| Anything else | -- | Running kernel vs. latest installed kernel |

An **unusable** answer is never read as "no reboot needed". If
`needs-restarting` is missing, or exits with anything other than `0`/`1`, the
probe reports `supported=false` and the host falls back to the kernel
comparison, with `reboot_required_source: kernel-comparison` and a
`reboot_reason` saying no distro check was available. The dashboard marks
those hosts "no distro check" under the Reboot column.

The Debian probe also reports the packages that asked for the reboot
(`reboot_pending_packages` / "Pending packages" in the host detail), read from
`/run/reboot-required.pkgs`.

## Latest installed kernel

The latest installed kernel is the version-highest entry under `/lib/modules`
that still looks like a real kernel -- it must have a `modules.dep`, or a
matching image in `/boot` (`vmlinuz-<version>`, `vmlinuz-<version>.efi`, or
`kernel-<version>`). A directory left behind by an interrupted package removal
therefore can't masquerade as the latest kernel. If nothing passes that filter
(minimal images, containers, unusual layouts), the probe falls back to the
plain `/lib/modules` listing rather than reporting no kernel at all.

`latest_kernel_selected` compares that version against the running kernel
(`ansible_facts['kernel']`).

## Bootloader default entry

The default boot entry is resolved in this order, stopping at the first source
that answers:

1. `grubby --default-kernel` (RHEL family) -- returns the kernel path directly.
2. `grub2-editenv`/`grub-editenv` + `grubenv` -- `saved_entry` is resolved as a
   BLS entry id, then a BLS entry title, then a `grub.cfg` menuentry title.
3. `/etc/default/grub`'s `GRUB_DEFAULT` -- resolved the same way when it names
   an entry, or by menuentry index when it's a number.
4. systemd-boot's `loader.conf` -- the `default` value (including a glob such
   as `fedora-*`, which resolves to the version-highest matching entry) is read
   from `loader/entries/`.

Resolved kernel paths are normalised before comparison: symlinks (Debian's
`/vmlinuz`) are followed, paths relative to a separate `/boot` partition are
resolved, and `vmlinuz-`/`linux-`/`kernel-` prefixes and a `.efi` suffix are
stripped.

Three statuses come out of this: `resolved` (a kernel version was determined),
`entry-only` (a default entry exists but its kernel couldn't be resolved), and
`unavailable` (no bootloader configuration was found). A finding fires only for
`resolved` entries that don't point at the latest installed kernel -- the case
where the newest kernel is installed but the host would boot an older one.

## Known edge cases

- **`needs-restarting` isn't installed.** RHEL/Fedora minimal images and
  containers often omit `dnf-utils`. The host reports
  `reboot_required_source: kernel-comparison`; install `dnf-utils` (or
  `yum-utils` on RHEL 7) for a precise answer.
- **`needs-restarting -r` reports a reboot for services, not just kernels.**
  A host can be flagged right after a `glibc`/`systemd` update with no kernel
  change at all. That is the tool's intent, not a false positive.
- **Old zypper.** `zypper needs-rebooting` only exists in zypper 1.14.44+.
  Older SUSE hosts silently fall through to the marker files, which are only
  written when the zypper plugin (or `transactional-update`) is installed.
- **Transactional/immutable hosts** (MicroOS, Silverblue, `rpm-ostree`) stage
  updates into a new snapshot. The marker files and `needs-restarting` describe
  the *booted* snapshot, so a pending snapshot may not be visible here.
- **Live-patched kernels** (kpatch, kgraft, Ubuntu Livepatch) keep the running
  kernel version older than the latest installed one on purpose. Those hosts
  report "latest installed kernel is not running" even though they are patched.
- **`/lib/modules` on containers.** Containers share the host kernel and often
  ship no modules at all; kernel currency findings there aren't meaningful.
- **Unified kernel images without a loader entry.** UKIs auto-discovered from
  `EFI/Linux/*.efi` (no `loader/entries/*.conf`) leave bootloader validation at
  `unavailable`. UKIs *with* an entry file resolve through its `efi` line.
- **Direct/EFI-stub and cloud boot paths.** Hosts with no GRUB and no
  systemd-boot configuration (many cloud images, `kexec`-booted hosts) report
  bootloader status `unavailable` -- expected, not an error.
- **`GRUB_DEFAULT=saved` without a readable `grubenv`.** `grub-editenv` needs
  read access to the environment block; without `become`, resolution can stop at
  `entry-only`.
- **`grub.cfg` submenus.** A numeric `GRUB_DEFAULT` counts top-level
  menuentries. A `1>2`-style submenu default resolves to `entry-only` rather
  than guessing.
- **Boot entries pointing at a not-yet-installed kernel.** If the default entry
  names a kernel whose `/lib/modules` directory is gone, `default_matches_latest`
  is `false` -- which is the correct alarm: that host would fail to boot.
