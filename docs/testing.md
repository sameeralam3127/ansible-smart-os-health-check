# Testing

LinuxVitals has two layers of automated tests, both run by CI on every push
and pull request:

| Layer | Command | What it proves |
| --- | --- | --- |
| Unit / template | `pytest -q` | Report templates render, reboot/kernel detection logic derives the right facts, and the embedded discovery shell scripts are POSIX-clean. No hosts involved. |
| Molecule scenarios | `molecule test -s <scenario>` | The roles actually run end to end against a live systemd host of each supported distribution. |

Plus `ansible-lint roles/ playbooks/ molecule/` and
`ansible-playbook playbooks/healthcheck.yml --syntax-check`.

## Molecule scenarios

One scenario per supported distribution family:

| Scenario | Image | Stands in for |
| --- | --- | --- |
| `ubuntu` | `geerlingguy/docker-ubuntu2404-ansible` | Debian family (Ubuntu, Debian) |
| `rocky` | `geerlingguy/docker-rockylinux9-ansible` | RHEL-compatible (RHEL, AlmaLinux, CentOS Stream) |
| `fedora` | `geerlingguy/docker-fedora42-ansible` | Fedora, and the newest dnf5/systemd behaviour ahead of RHEL |
| `opensuse` | `dokken/opensuse-leap-15` | SUSE family (openSUSE Leap, SLES) |

Each one boots a container with **systemd as PID 1** (privileged, host cgroup
namespace) -- `vitals_scan` reads `service_facts` and `journalctl`, so a
sleep-forever container would exercise none of the paths worth testing.

### Running them

Requires a working Docker daemon. Everything else is installed by
`pip install -r requirements-dev.txt`.

```bash
molecule test -s ubuntu          # one distribution, full create/converge/verify/destroy
molecule test --all              # every scenario, sequentially

molecule converge -s rocky       # leave the container up for debugging
molecule login -s rocky          # shell into it
molecule verify -s rocky         # re-run only the assertions
molecule destroy -s rocky        # clean up
```

The generated dashboard, JSON report, and archive for a run land in that
scenario's ephemeral directory (`molecule converge` prints the path; it is
`~/.ansible/tmp/molecule.*.<scenario>/reports/`).

### What a scenario does

1. **create** -- starts the container and ensures
   `.dev-collections/ansible_collections/sameeralam3127/linux_vitals` links back
   to the repo, so `converge.yml`'s FQCN roles resolve from a fresh clone.
2. **prepare** -- bootstraps a Python interpreter Ansible can use, installs the
   packages the checks look for (chrony, and `yum-utils` on Rocky so
   `needs-restarting` exists), then plants two systemd units:
   - `molecule-flaky.service`, which fails on first start and succeeds on
     restart -- a service `vitals_heal` can genuinely fix;
   - `molecule-broken.service` (`ExecStart=/bin/false`), which can never be
     fixed.
   Both are enabled and left in a failed state.
3. **converge** -- runs `vitals_scan` -> `vitals_heal` -> `vitals_report` with
   `linux_vitals_heal_enabled: true`, writing reports into the scenario's
   ephemeral directory instead of `inventory_dir`.
4. **verify** -- asserts against the generated JSON report, the HTML dashboard,
   and the container itself:
   - **discovery**: OS family, package manager, log source, uptime, memory,
     running kernel, boot-space status, security-control status, and that the
     time-sync service resolved and is active;
   - **reboot detection**: the source is one this distribution can actually
     provide, so a regression that silently degrades to the kernel-comparison
     fallback fails the scenario (see
     [kernel-reboot-detection.md](kernel-reboot-detection.md));
   - **self-healing**: `molecule-flaky.service` is reported `Fixed`, is counted
     in `auto_fixed_count`, and is genuinely `active` on the host;
     `molecule-broken.service` is *not* fixed and raises a
     "requires manual follow-up" finding;
   - **reporting**: the JSON summary, the rendered dashboard, and the
     timestamped archive copy.

Each distribution reaches reboot detection by a different route, which is the
point of running all four:

| Scenario | `reboot_required_source` |
| --- | --- |
| `ubuntu` | `reboot-required-file` |
| `rocky` | `needs-restarting` |
| `fedora` | `dnf-needs-restarting` (dnf5) |
| `opensuse` | `zypper-needs-rebooting` |

### The SUSE image (fallback strategy)

There is no systemd + Ansible-ready openSUSE image from the same publisher as
the other three, so the SUSE scenario uses Chef's `dokken/opensuse-leap-15`,
which runs systemd as PID 1 but ships **no Python at all**. Leap 15.6's default
`python3` is 3.6, which ansible-core cannot use, so the scenario installs
`python311` over `raw` and pins `ansible_python_interpreter` to
`/usr/bin/python3.11`. This is per-scenario configuration
(`vitals_python_package` / `vitals_python_interpreter` in
`molecule/opensuse/molecule.yml`), so swapping in a different SUSE image later
means changing two lines.

If the dokken image ever becomes unavailable, the fallbacks in order of
preference are: build a small `Dockerfile` from `opensuse/leap:15.6` that
installs `systemd` and `python311`; or drop to `opensuse/tumbleweed`, accepting
that it tracks a rolling release rather than the SLES-aligned Leap.

### What containers cannot prove

Containers share the host kernel and have no bootloader, so some findings fire
in every scenario and are deliberately not asserted against:

- `Latest installed kernel is not currently running` -- `/lib/modules` is empty
  in a container, so no installed kernel can match `uname -r`.
- Bootloader validation reports `unavailable` -- there is no `/boot` content,
  no GRUB, and no systemd-boot loader configuration.
- `boot_space_status` is `Not Available` -- `/boot` is not a separate mount.
- `RAM usage is critical` can fire depending on the Docker host's memory
  pressure at the time.

Bootloader resolution and latest-kernel filtering are covered instead by the
synthetic-root unit tests in `tests/test_reboot_detection.py`, which run the
real discovery shell scripts against a fabricated `/boot` layout.
