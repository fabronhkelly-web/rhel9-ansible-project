# RHEL 9 Ansible Lab: baseline, patching, users and DISA STIG

Home lab automation for Red Hat Enterprise Linux 9.9 servers, built to practice
DoD-style system administration: configuration baselines, rolling security
patching, account management and DISA STIG compliance verified with OpenSCAP.

## Lab layout
| Host | Role |
|---|---|
| rhel01.lab.example.com | Ansible control node (also managed by the baseline) |
| rhel02.lab.example.com | Managed server (target of patching and the STIG playbook) |

Lab domain is `lab.example.com` (reserved for documentation). No real hostnames or IPs.

## What is here
| Path | Purpose |
|---|---|
| `ansible.cfg` | Project settings: `ansible` service account, key auth, sudo, Vault password file |
| `inventory/hosts.ini` | Hosts grouped as `control`, `managed`, `rhel9` |
| `requirements.yml` | Pinned collections (`ansible.posix` 1.5.4 for ansible-core 2.14) |
| `playbooks/baseline.yml` | Packages, chrony, auditd, DoD banner, SSH hardening, firewalld (idempotent) |
| `playbooks/patch.yml` | Rolling security-only patching, one host at a time, reboot only if `needs-restarting -r` says so |
| `roles/lab_users` | Admin group, users with 60/1-day password aging, validated sudo rule |
| `playbooks/breakglass.yml` | Break-glass local admin, password from Ansible Vault (`no_log`) |
| `playbooks/rhel9-playbook-stig.yml` | Red Hat `scap-security-guide` DISA STIG playbook, scoped to `managed` |

## How to run
```bash
ansible-galaxy collection install -r requirements.yml
ansible all -m ping
ansible-playbook playbooks/baseline.yml --check --diff   # dry run first
ansible-playbook playbooks/baseline.yml                  # apply; rerun shows changed=0
ansible-playbook playbooks/patch.yml
ansible-playbook playbooks/users.yml
```
Vault file `inventory/group_vars/rhel9/vault.yml` and the vault password are excluded from Git.

## Design notes
- **Service account:** dedicated `ansible` user, key-only (password locked), scoped sudo.
- **FIPS mode:** hosts run in FIPS mode with the `FIPS:STIG` crypto policy, so keys are ECDSA (Ed25519 is not allowed in FIPS).
- **Drift detection:** cron runs `baseline.yml --check --diff` nightly; a simulated change to SSH and banner permissions was detected and corrected.
- **Safety:** `serial: 1` and `max_fail_percentage: 0` in patching so one bad patch stops the run.

## STIG results (OpenSCAP, DISA STIG profile for RHEL 9)
| Host | Stage | Score | Failed rules | CAT I failed |
|---|---|---|---|---|
| rhel01 | Fresh install | 47.4% | 264 | 10 |
| rhel01 | Manual fixes (banner, SSH, pwquality, mounts, audit rules, PAM, Ctrl-Alt-Del, GPG) | 51.3% | 207 | 5 |
| rhel01 | + FIPS mode and FIPS:STIG crypto policy | 51.7% | 200 | 2 |
| rhel01 | + GRUB2 password and non-default superuser (`bootadmin`) | 52.1% | 199 | **0** |
| rhel02 | Clone of hardened rhel01 (before playbook) | 52.3% | 198 | 0 |
| rhel02 | After `rhel9-playbook-stig.yml` (46 min, ok=2999 changed=350 failed=0) | **95.2%** | **14** | **0** |

All 10 CAT I findings on rhel01 are closed. The GRUB superuser rename had no automated fix in the SCAP content, so it was done by hand in `/etc/grub.d/01_users`, then verified with a reboot. The remaining CAT II/III findings are targeted by the STIG playbook; exceptions are tracked in a POA&M-style table.

### Remaining findings on rhel02 (14 CAT II, tracked in the POA&M)
- **Skipped on purpose** so automation keeps working: `sudo_remove_nopasswd`, `service_fapolicyd_enabled`, `fapolicy_default_deny`
  (run with `--skip-tags`; in production the Ansible account would use Ansible Automation Platform credentials with a sudo password,
  and fapolicyd would get a trust rule for Ansible before enforcing).
- **Lab environment:** single DNS server (IdM adds a second in Project 3), smart card certmap (no PKI), chrony `pool` vs `server`,
  authorized local user list, user namespaces (no automated fix), SELinux sudo context elevation, `pam_wheel` for `su`,
  world-writable directory without sticky bit (the lab's VMware shared folder), SSH MAC crypto-policy and `difok` checks.

```bash
SKIP=sudo_remove_nopasswd,sudo_remove_no_authenticate,service_fapolicyd_enabled,fapolicy_default_deny
ansible-playbook playbooks/rhel9-playbook-stig.yml --skip-tags "$SKIP"
ansible managed -m ansible.builtin.reboot
```
