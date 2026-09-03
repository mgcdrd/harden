# harden deployment

Post-build OS hardening (CIS Level 1/2) via `mgcdrd.infrabase` roles. Run once
after provisioning, before handing a host off to a service deployment
(`foreman`, `keycloak`, `k8s`, etc.) — those assume the host is already
hardened and do not re-apply these controls themselves.

---

## Role execution order

| Phase | Roles | CIS area |
|---|---|---|
| Storage | `lvm2` | n/a — OS partition/LV expansion, not a CIS control |
| Proxmox guest agent | `qemu_guest_agent` | n/a — VM/Proxmox integration, not a CIS control. Skips `proxmox_ve` and `misc` (not QEMU guests), and self-skips inside a container |
| Kernel and filesystem | `kernel_modules`, `sysctl`†, `secure_mounts`†, `coredump` | 1.1, 3.1–3.4 |
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
though — `ansible.cfg`'s third inventory source (`build.foreman.yml`) needs
`FOREMAN_URL`/`FOREMAN_USER`/`FOREMAN_PASSWORD` in the environment before
running. `source ../../inventory-common/foreman-inventory-env.sh` (Vault-
backed) before `ansible-playbook` if you're targeting anything on the
`building` group; skip it entirely if you're only targeting hosts already on
their real network — the dynamic source just returns nothing.

---

## Container / LXC targets

The playbook runs against a Rocky 9/10 or Debian 12/13 **container** (Proxmox
LXC, `systemd-nspawn`, Docker) as well as a full VM. The storage play sets
`harden_container_guest` from `ansible_facts['virtualization_type']` /
`virtualization_role`, and four roles are skipped in a container via per-role
`harden_container_skip_<role>` toggles (each defaults to
`harden_container_guest`):

| Skipped role | Toggle | Why |
|---|---|---|
| `sysctl` | `harden_container_skip_sysctl` | `kernel.*` / `fs.*` keys are read-only from the container namespace; `sysctl --system` would fail |
| `secure_mounts` | `harden_container_skip_secure_mounts` | remounting `/dev/shm` returns `EPERM` in an unprivileged container; mount opts are inherited from the host |
| `firewall` | `harden_container_skip_firewall` | `firewalld` / `nftables` can't manage host netfilter from an unprivileged container — filter at the host or in Proxmox |
| `auditd` | `harden_container_skip_auditd` | the kernel audit subsystem isn't namespaced; the host's `auditd` already records container activity |

Everything else (`lvm2`, `kernel_modules`, `coredump`, `login_banner`, `sshd`,
`password_policy`, `pam`, `sudoers`, `cron`, `hardened_services`,
`crypto_policies`, `postfix`, `journald`, `rsyslog`, `aide`) runs unchanged.
`qemu_guest_agent` self-skips inside a container regardless of this deployment.

### Running one of the four on a container that supports it

Override just that toggle in `inventory/host_vars/<fqdn>.yml`. The common case
is a container that keeps its **own** firewall — a privileged LXC, or an
unprivileged one on a host whose kernel has `nf_tables` loaded. Confirm it
works in the container first:

```bash
nft add table inet t && nft delete table inet t && echo "nftables OK"
systemctl start firewalld && firewall-cmd --state
```

then:

```yaml
# inventory/host_vars/ct-web01.example.com.yml
harden_container_skip_firewall: false
```

`sysctl` / `secure_mounts` / `auditd` on that host stay skipped. Note Proxmox's
own CT/bridge firewall is a separate layer in front of the container's veth —
running both is fine but is two rulesets to maintain.

---

## Prerequisites

- Rocky 9/10 or Debian 12/13 VM or container, reachable via SSH with `become: true`
- Add the host to `../../inventory-common/hosts.yml`, under its service-role
  group (e.g. `foreman`) — this playbook targets `hosts: all`, so
  every host defined there gets hardened. A host still on Foreman's Build
  subnet doesn't need adding here at all — `build.foreman.yml` picks it up
  automatically into the `building` group, reachable via SSH ProxyJump
  through Foreman (see `../../inventory-common/README.md`)
- If the host is (or will be) IPA-enrolled, do that enrollment **before**
  running this playbook — changing the `pam_authselect_profile` afterward can
  break LDAP auth

---

## Usage

```bash
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml
```

No tags are defined per-phase — every run applies the full control set. All
roles are idempotent and safe to re-run.

---

## Key variables

Hardening tuning specific to this deployment lives in
`inventory/group_vars/all/main.yml`; per-host overrides go in
`inventory/host_vars/<fqdn>.yml`. Host/group topology, `ansible_user`, and
`rsyslog_remote_host` come from `../../inventory-common` instead — see that
repo's README for the tier rule on what belongs where.

| Variable | Default | Notes |
|----------|---------|-------|
| `harden_container_guest` | auto-detected (`false` fallback) | Set true by the storage play on LXC/nspawn/Docker guests |
| `harden_container_skip_{sysctl,secure_mounts,firewall,auditd}` | `harden_container_guest` | Per-role container skip. Override one `false` in host_vars to run that role on a container that supports it — see [Container / LXC targets](#container--lxc-targets) |
| `kernel_modules_blacklist` | unused net protocols, uncommon FS, usb-storage | Remove `usb-storage` if the host needs USB mass storage |
| `secure_mounts_list` | `/dev/shm` hardened | `/tmp` entry is commented out — only enable if `/tmp` is a dedicated partition |
| `sshd_allow_groups` | unset | Uncomment to restrict SSH to specific groups |
| `pam_authselect_profile` | `sssd` | Do not change on IPA-enrolled hosts |
| `hardened_services_disable_nfs` | `true` | Set `false` on intentional NFS server hosts |
| `lvm_pv_grow` | `[]` | Opt-in per host — grows a partition + its PV to consume space added to the underlying disk. See `mgcdrd.infrabase.lvm2`'s README |
| `lvm_volumes` | `[]` | Opt-in per host — extends specific LVs (e.g. `lv_root`) once `lvm_pv_grow` (or a VG extension) has made room |

`firewall_zones` (in `inventory-common/group_vars/<group>.yml`) and
`rsyslog_remote_host` (in `inventory-common/group_vars/all.yml`) are no
longer set here — see `../../inventory-common/README.md`.

---

## Client/customer delivery

Portable as-is: point `ansible.cfg`'s inventory path at the customer's
`inventory-<client>` repo (cloned from `inventory-template`) instead of the
lab's `inventory-common`, and swap this deployment's own `inventory/`
overrides for the customer's hosts. The roles themselves have no Vault
dependency — only `foreman` (or whichever service deployment follows) needs
Vault populated for its own secrets. If the customer's Foreman also gateways
a NAT'd build network, `build.foreman.yml`'s credential needs populating in
their Vault too (see `inventory-<client>/foreman-inventory-env.sh`,
mirrored from `inventory-template`); if they don't use that pattern, drop
the third `-i` source from `ansible.cfg` and the `theforeman.foreman`
collection dependency.
