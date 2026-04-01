# Ansible role: `ahmz.server_setup.core`

Baseline **Linux server provisioning**: identity (hostname, timezone), packages, optional swap, users and SSH hardening, **sysctl**, **ipset**, **iptables** INPUT firewall, **fail2ban**, and managed **cron** jobs.

The role is built for **systemd**-based hosts. It works best on **Debian/Ubuntu** and **Red Hat family** (incl. Fedora-style); several features are **family-specific** (see below).

---

## Requirements

| Requirement | Notes |
|-------------|--------|
| **Ansible** | 2.14+ (`meta/main.yml`). |
| **Collections** | `community.general` (e.g. `timezone`), `ansible.posix` (`mount`, `authorized_key`). Declared in role `meta/main.yml`. |
| **Target** | Linux with **systemd**, **Python** for Ansible modules, and privilege escalation where tasks use `become`. |
| **Facts** | `gather_facts: true` recommended (`ansible_facts` for OS family, dates, `getent`, etc.). |

---

## What runs where (OS family)

| Area | Debian family | Red Hat family | Other (e.g. Alpine, partial) |
|------|----------------|----------------|------------------------------|
| **APT mirrors** | Yes (`apt` sources rewrite when `core_set_apt_mirrors`) | No-op | No-op |
| **`apt update`** | Yes | No | No |
| **Packages** | `ansible.builtin.package` (names must exist in that distro’s repos) | Same | Override `core_packages` if names differ |
| **NTP** | `systemd-timesyncd` | **Chrony** (also **SUSE**, **Arch Linux**) | Not configured — set `core_manage_ntp: false` or extend locally |
| **Firewall save** | `netfilter-persistent save` | `/etc/sysconfig/ip*tables` + `iptables` service | **Rules apply in memory; not saved** on reboot unless you add your own persistence |
| **SSH service** | `ssh` | `sshd` | Same split by `os_family == 'Debian'` |

---

## Role entrypoint and tags

`tasks/main.yml` order:

1. **system** — timezone, hostname, optional `resolv.conf` / `hosts`
2. **packages** — APT mirrors (Debian), cache update, `core_packages` + `core_extra_packages`
3. **ntp** — if `core_manage_ntp`
4. **swap** — if `core_manage_swap`
5. **users** — if `core_manage_users`
6. **ssh** — if `core_manage_ssh`
7. **sysctl** — if `core_manage_sysctl`
8. **ipset** — if `core_manage_ipset` and `core_ipsets` non-empty
9. **firewall** — if `core_manage_firewall`
10. **fail2ban** — if `core_fail2ban_enabled`
11. **cron** — if `core_manage_cron`

| Tag | Scope |
|-----|--------|
| `core` | Entire role |
| `core-system` | system + packages |
| `core-packages` | packages |
| `core-ntp` | NTP |
| `core-swap` | Swap |
| `core-users` | Users |
| `core-ssh` | SSH (+ users tag overlap) |
| `core-sysctl` | Sysctl |
| `core-ipset` | IPSet (+ cron auto-ipset) |
| `core-firewall` | Firewall |
| `core-fail2ban` | Fail2ban |
| `core-cron` | Cron |

---

## Playbook-level meta variables

These are usually set in the playbook and read via `defaults` (`| default(...)`):

| Variable | Role default usage |
|----------|---------------------|
| `is_iran` | Drives `core_is_iran` → timezone (`Asia/Tehran` vs `Etc/UTC`) and APT mirror toggle |
| `enable_ipv6` | `core_enable_ipv6` → sysctl + firewall IPv6 block |
| `download_locally` | Passed through as `core_download_locally` for other roles; not heavily used inside core |
| `primary_user` | `core_primary_user` — protected from user purge; SSH key checks |

---

## Notable defaults (see `defaults/main.yml` for the full list)

### Toggles

`core_manage_ntp`, `core_manage_swap`, `core_manage_users`, `core_manage_ssh`, `core_manage_sysctl`, `core_manage_ipset`, `core_manage_firewall`, `core_manage_cron`, `core_fail2ban_enabled` — each can be set `false` to skip that block.

### Packages

`core_packages` uses **Debian-style names** (e.g. `netcat-openbsd`). On **RHEL**, override the list or use `core_extra_packages` / replace with distro-specific names (`nmap-ncat`, etc.).

### Sysctl and BBR

`core_sysctl_base` sets **`net.ipv4.tcp_congestion_control: bbr`** and **`net.core.default_qdisc: fq`**, which need a kernel with **BBR**. If `sysctl --system` fails on your image, set:

```yaml
core_sysctl_enable_bbr: false
```

You can still tune congestion control in `core_sysctl_extra` for a module your kernel provides.

### SSH

`core_ssh_port` resolves from `ansible_facts['port']` / `ansible_facts['ssh_port']`, then `ssh -G <inventory_hostname>`, then `22`. Ensure inventory or SSH config matches reality before hardening.

### Swap

With `core_swap_enabled: false`, the disable path runs **`swapoff -a`** (all swap), not only the managed file — intentional for “no swap” servers; be careful if you use multiple swap devices.

### `resolv.conf`

If `core_manage_nameservers` is true, the role may **remove a symlink** and write a static file. On hosts using **systemd-resolved**, reconcile with your DNS strategy.

### IPSet / firewall

- **ipset** uses `systemd` unit **`ipset-restore`** for boot-time restore.
- **Firewall** uses **`ansible.builtin.iptables`** (legacy **iptables** backend). Distros defaulting to **nftables** only may still ship the `iptables` compatibility tools — verify on minimal images.

---

## Example playbook

```yaml
- name: Baseline servers
  hosts: all
  become: true
  vars:
    primary_user: admin
    enable_ipv6: false

    core_users:
      - name: admin
        groups: ["sudo", "docker"]
        shell: /bin/bash
        ssh_keys:
          - "{{ lookup('env', 'HOME') }}/.ssh/id_ed25519.pub"
        sudoer: true
        sudoer_no_pass: true

    core_firewall_rules:
      - { port: 80, comment: "HTTP" }
      - { port: 443, comment: "HTTPS" }

    core_ipsets:
      - name: office-nets
        type: hash:net
        entries:
          - "10.0.0.0/8"

  roles:
    - role: ahmz.server_setup.core
```

---

## Handlers

- **Reload sysctl** — `sysctl --system`
- **Restart** `fail2ban`, `sshd`/`ssh`, `timesyncd`, `chronyd`, `ipset-restore`
- **systemd daemon-reload**

---

## License

MIT
