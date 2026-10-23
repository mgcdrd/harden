# harden deployment

Post-build OS hardening (CIS Level 1/2) via `mgcdrd.infrabase` roles. Run once
after provisioning, before handing a host off to a service deployment
(`foreman`, `keycloak`, `k8s`, etc.) — those assume the host is already
hardened and do not re-apply these controls themselves.

---

## Role execution order

| Phase | Roles | CIS area |
|---|---|---|
| Ansible remote tmp bootstrap | (raw task, no role) | n/a — creates `/var/lib/ansible-tmp` so later plays survive `storage` remounting `/home` noexec |
| Storage | `storage`† | n/a — OS partition/LV expansion and mounts; the mount options below are CIS 1.1 |
| Proxmox guest agent | `qemu_guest_agent` | n/a — VM/Proxmox integration, not a CIS control. Skips `proxmox_ve` and `misc` (not QEMU guests), and self-skips inside a container |
| Kernel and filesystem | `kernel_modules`, `sysctl`†, `coredump` | 1.1, 3.1–3.4 |
| Access control | `login_banner`, `sshd`, `password_policy`, `pam`, `sudoers`, `cron` | 1.7, 5.2–5.4 |
| Services | `hardened_services`, `firewall`†, `crypto_policies` | 2.x, 3.4–3.5 |
| Mail relay client | `postfix` | n/a — outbound relay for cron/alert mail, not a CIS control |
| Logging and auditing | `journald`, `rsyslog`, `auditd`† | 4.1, 4.2 |
| File integrity | `aide` | 1.3 |

`aide` runs last so its database captures the final hardened state rather
than a pre-hardening snapshot.

† Skipped on container guests — see [Container / LXC targets](#container--lxc-targets).

---

## Vault

**None required for the roles themselves** — this deployment applies
OS-level controls only (kernel, sysctl, PAM, sudo, firewalld, logging,
AIDE), no `vault.yml` or Vault lookups anywhere in this deployment or the
roles it consumes.

Reaching hosts still on Foreman's build network *does* need a credential,
though — `ansible.cfg`'s second inventory source (`build.foreman.yml`) needs
`FOREMAN_URL`/`FOREMAN_USER`/`FOREMAN_PASSWORD` in the environment before
running. `source ../../inventory-common/lab/foreman-inventory-env.sh` (Vault-
backed) before `ansible-playbook` if you're targeting anything on the
`building` group; skip it entirely if you're only targeting hosts already on
their real network — the dynamic source just returns nothing.

---

## Container / LXC targets

The playbook runs against a Rocky 9/10 or Debian 12/13 **container** (Proxmox
LXC, `systemd-nspawn`, Docker) as well as a full VM. The storage play sets
`harden_container_guest` from `ansible_facts['virtualization_type']` /
`virtualization_role`, and four roles are skipped when it is true:

| Skipped role | Why |
|---|---|
| `sysctl` | `kernel.*` / `fs.*` keys are read-only from the container namespace; `sysctl --system` would fail |
| `storage` | remounting `/dev/shm` returns `EPERM` in an unprivileged container; mount opts are inherited from the host |
| `firewall` | `firewalld` / `nftables` can't manage host netfilter from an unprivileged container — filter at the host or in Proxmox |
| `auditd` | the kernel audit subsystem isn't namespaced; the host's `auditd` already records container activity |

Everything else (`kernel_modules`, `coredump`, `login_banner`, `sshd`,
`password_policy`, `pam`, `sudoers`, `cron`, `hardened_services`,
`crypto_policies`, `postfix`, `journald`, `rsyslog`, `aide`) runs unchanged.
`qemu_guest_agent` self-skips inside a container regardless of this deployment.

### Forcing the container skips off

`harden_container_guest` is detected with `set_fact`, so it can't be overridden
from `host_vars`. For a container that can run these roles (a privileged LXC),
pass `-e harden_container_guest=false` with `--limit <host>`. That runs all four
roles on it, so confirm each works in the container first. Proxmox's own
CT/bridge firewall is a separate layer in front of the container's veth;
running both is fine but is two rulesets to maintain.

---

## Prerequisites

- Rocky 9/10 or Debian 12/13 VM or container, reachable via SSH with `become: true`
- Add the host to the environment's `hosts.yml` (`../../inventory-common/lab/hosts.yml` for the lab), under its service-role
  group (e.g. `foreman`) — this playbook targets `hosts: all`, so
  every host defined there gets hardened. A host still on Foreman's Build
  subnet doesn't need adding here at all — `build.foreman.yml` picks it up
  automatically into the `building` group, reachable via SSH ProxyJump
  through Foreman (see `../../inventory-common/README.md`)
- If the host is (or will be) IPA-enrolled, do that enrollment **before**
  running this playbook — changing the `pam_authselect_profile` afterward can
  break LDAP auth

## Variable reference

All variables live in the environment inventory (`inventory-common/lab` for the lab): fleet-wide hardening policy in `group_vars/all/harden.yml`, per-group exceptions in `group_vars/<group>/harden.yml`, per-host overrides in `host_vars/<fqdn>/`. Only values that differ from the role defaults are set; anything not listed keeps the role README defaults, which are already CIS-aligned.

### Set in `group_vars/all/harden.yml`

| Variable | Shipped value | Required | Description |
|---|---|---|---|
| `harden_container_guest` | `false` | No | Fallback flag for `--limit` runs that skip the storage play; that play sets it true on LXC, nspawn, and Docker guests |
| `kernel_modules_blacklist` | unused network protocols (`dccp`, `sctp`, `rds`, `tipc`), uncommon filesystems (`cramfs`, `freevxfs`, `jffs2`, `hfs`, `hfsplus`, `squashfs`, `udf`), `usb-storage` | No | Modules blocked from loading (CIS 1.1); remove `usb-storage` if a host needs USB mass storage |
| `sysctl_conf_file` | `/etc/sysctl.d/99-hardening.conf` | No | File the parameters are written to |
| `sysctl_params` | IPv4/IPv6 routing hardening, `kernel.randomize_va_space: 2`, `kernel.dmesg_restrict: 1`, `kernel.perf_event_paranoid: 3`, `kernel.yama.ptrace_scope: 1`, `fs.suid_dumpable: 0` | No | Network and process hardening (CIS 3.1 to 3.3); override `net.ipv4.ip_forward` for routers and k8s nodes |
| `login_banner_text` | generic authorized-use notice | No | Pre-auth banner (CIS 1.7); should not include hostname, OS, or version |
| `sshd_banner` | `/etc/issue.net` | No | Banner file, written by `login_banner` |

The `pam`, `hardened_services`, `journald`, `sudoers`, and `password_policy` roles run on their own defaults. Adjust them by setting the role's variable (e.g. `pam_authselect_features`, `hardened_services_disable_nfs`, `password_policy_login_defs`) in the inventory. `storage_mounts` and the `rsyslog_remote_*` / `postfix_relayhost` values are set in the same file; `sshd_allow_groups` can be added per group (use `sshd_allow_groups_extra_<suffix>` to add to a base list).

### Per-host opt-ins (`host_vars/<fqdn>/`)

| Variable | Default | Required | Description |
|---|---|---|---|
| `storage_scheme` | `""` | No | Build-time layout profile (`std`, `base-lvm`, `ext-lvm`, `foreman-el-uefi`); see `mgcdrd.infrabase.storage` |
| `storage_disks` | `[]` | No | Grow a partition and its PV into space added to the underlying disk, or add disks to a VG |
| `storage_mounts` | from `all` | No | Override per host, e.g. add dedicated `/home` and `/var` mounts with `nodev`/`nosuid`; use `lv` and `size` to extend an LV |
| `hardened_services_fix_unowned_files` | `false` | No | `chown` unowned files to `root:root`; enable per host after reviewing what it would touch |

### Environment

| Variable | Required | Description |
|---|---|---|
| `FOREMAN_URL`, `FOREMAN_USER`, `FOREMAN_PASSWORD` | Only for hosts in the Foreman `building` group | Used by the second inventory source (`build.foreman.yml`); load them with `source ../../inventory-common/lab/foreman-inventory-env.sh` (Vault-backed) |

---

## Usage

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml                       # every host in the environment inventory
ansible-playbook site.yml --limit <host>        # one host or group
```

`ansible.cfg` points at the lab environment (`inventory-common/lab/hosts.yml`).
For another environment pass `-i <inventory>/<env>/hosts.yml`.

No tags are defined per-phase — every run applies the full control set. All
roles are idempotent and safe to re-run.

---

## Client/customer delivery

Portable as-is: point `ansible.cfg`'s inventory path at the customer's
`inventory-<client>` repo (cloned from `inventory-template`) instead of the
lab's `inventory-common`; the customer's hosts and overrides live in that
repo's environment directory. The roles themselves have no Vault
dependency — only `foreman` (or whichever service deployment follows) needs
Vault populated for its own secrets. If the customer's Foreman also gateways
a NAT'd build network, `build.foreman.yml`'s credential needs populating in
their Vault too (see `inventory-<client>/foreman-inventory-env.sh`,
mirrored from `inventory-template`); if they don't use that pattern, drop
the `build.foreman.yml` source from `ansible.cfg` and the `theforeman.foreman`
collection dependency.
