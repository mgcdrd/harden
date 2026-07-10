# harden deployment

Post-build OS hardening (CIS Level 1/2) via `mgcdrd.infrabase` roles. Run once
after provisioning, before handing a host off to a service deployment
(`foreman`, `keycloak`, `k8s`, etc.) — those assume the host is already
hardened and do not re-apply these controls themselves.

---

## Role execution order

| Phase | Roles | CIS area |
|---|---|---|
| Kernel and filesystem | `kernel_modules`, `sysctl`, `secure_mounts`, `coredump` | 1.1, 3.1–3.4 |
| Access control | `login_banner`, `sshd`, `password_policy`, `pam`, `sudoers`, `cron` | 1.7, 5.2–5.4 |
| Services | `hardened_services`, `firewall`, `crypto_policies` | 2.x, 3.4–3.5 |
| Logging and auditing | `journald`, `rsyslog`, `auditd` | 4.1, 4.2 |
| File integrity | `aide` | 1.3 |

`aide` runs last so its database captures the final hardened state rather
than a pre-hardening snapshot.

---

## Vault

**None required.** This deployment applies OS-level controls only (kernel,
sysctl, PAM, sudo, firewalld, logging, AIDE) — nothing it manages needs a
credential, so no `vault.yml` or Vault lookups exist anywhere in this
deployment or the roles it consumes.

---

## Prerequisites

- Rocky 9/10 or Debian 12/13 VM, reachable via SSH with `become: true`
- Add the host to the `hardened` group in `inventory/hosts.yml`
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

All variables are in `inventory/group_vars/all/main.yml`; per-host overrides
go in `inventory/host_vars/<fqdn>.yml`.

| Variable | Default | Notes |
|----------|---------|-------|
| `kernel_modules_blacklist` | unused net protocols, uncommon FS, usb-storage | Remove `usb-storage` if the host needs USB mass storage |
| `secure_mounts_list` | `/dev/shm` hardened | `/tmp` entry is commented out — only enable if `/tmp` is a dedicated partition |
| `sshd_allow_groups` | unset | Uncomment to restrict SSH to specific groups |
| `pam_authselect_profile` | `sssd` | Do not change on IPA-enrolled hosts |
| `hardened_services_disable_nfs` | `true` | Set `false` on intentional NFS server hosts |
| `firewall_zones` | SSH-only `public` zone | Add ports/services here, or in host_vars for host-specific exceptions |
| `rsyslog_remote_host` | — | Set to ship logs to a central syslog server; leave empty to manage file perms only |

---

## Client/customer delivery

Portable as-is: swap `inventory/hosts.yml` and `group_vars`/`host_vars` for
the customer's hosts and network values. Since there's no Vault dependency,
there's nothing to provision in their secrets backend for this deployment —
only `foreman` (or whichever service deployment follows) needs Vault
populated.
