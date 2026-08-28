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
| Kernel and filesystem | `kernel_modules`, `sysctl`, `secure_mounts`, `coredump` | 1.1, 3.1–3.4 |
| Access control | `login_banner`, `sshd`, `password_policy`, `pam`, `sudoers`, `cron` | 1.7, 5.2–5.4 |
| Services | `hardened_services`, `firewall`, `crypto_policies` | 2.x, 3.4–3.5 |
| Logging and auditing | `journald`, `rsyslog`, `auditd` | 4.1, 4.2 |
| File integrity | `aide` | 1.3 |

`aide` runs last so its database captures the final hardened state rather
than a pre-hardening snapshot.

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

## Prerequisites

- Rocky 9/10 or Debian 12/13 VM, reachable via SSH with `become: true`
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
