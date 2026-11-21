# CLAUDE.md – working notes for AI assistants (and humans) in this repo

This file is read at the start of every Claude Code session. It holds **how we
work here and why things are the way they are**, so that the same lessons do
not have to be learned twice. It is public: no hostnames, addresses, provider
names, personal data or secrets – those stay in the gitignored inventory and
`docs/intern/`.

The long explanations live where the setting lives: in the comments of the
roles (look for "found 2026-…"), in `docs/` (operations guides in DE/EN/FR)
and in the README. This file only points the way and lists the traps.

## Layout

- Playbooks: `nextcloud-k3s.yml` (Nextcloud hosts), `nextcloud-update.yml`
  (updates, one host at a time).
- Roles: `common_*` (base), `next_*` (Nextcloud).
- Variables: `inventory/group_vars/all/main.yml`, pinned versions in
  `inventory/group_vars/all/versions.yml`. Real `host_vars` are gitignored;
  `*.example` files show the structure.
- Backup scripts: `scripts/nextcloud-backup/` (live copies run on the backup
  machines; keep both in sync).
- History: split from `www_k3s` (archived, tag `final-before-split`). The blog has
  its own repository; hardening fixes are not shared automatically, so check
  whether a security fix here also applies there.

## How we work

- **Lint the whole repo** before committing: `ansible-lint` (the CI runs it on
  everything, not per role). Pre-commit runs gitleaks, yamllint, ansible-lint.
- **Check mode first** (`--check --diff`), then the real run, **one host at a
  time** and with `-l <host>` (parallel runs on one host collided on the rpm
  lock). Afterwards a drift check (`scripts/check-drift.sh` or the same tags in
  `--check`) must show `changed=0`.
- **After any `--tags` run, also run `--tags common_rkhunter,common_aide_refresh`**,
  otherwise AIDE and rkhunter mail about the changed files at night.
- **Secret check before every commit and push**: read the full diff, grep for
  passwords/keys/IPs. Never print vault contents; mask passwords in command
  output (`MYSQL_PWD`, `REDISCLI_AUTH`, `sed` the `password`/`secret` lines).
- **No push without an explicit OK** from the repository owner.
- **Fix stale comments and docs immediately** when found; docs exist in three
  languages (`docs/*-operations|-betrieb|-exploitation.html`). Each HTML doc also exists as
  `docs/*.pdf`, rendered from it (headless Chromium, A4): the HTML is the source,
  re-render the PDFs after every change so they never go stale.
- **Delete only what is certainly dead** – data of disabled apps stays; back up
  first, count live rows.

## Traps we already fell into (and the rule that came out of it)

### Updates
- **dnf-automatic with `upgrade_type = security` never touches vendor repositories** (Grafana, Rancher
  k3s-selinux, MariaDB, nginx.org on the blog): they publish no updateinfo, so nothing in them counts as a
  security fix (found 2026-11-20). `update-vendor-repos` (common_dnf_automatic, list in
  `dnf_automatic_vendor_repos`) takes every update from the listed repos at 02:15, MariaDB included (only
  patch releases of the series pinned in the repo; mariadbd is restarted at 03:45 - check `ps -eZ | grep mariadbd`
  shows mysqld_t after the first automatic update).

### SELinux
- **NoNewPrivileges blocks domain transitions.** Under NNP the policy only
  allows a transition into a domain that grants `nnp_transition`. MariaDB
  started via `/bin/sh` stayed in `initrc_t` for five weeks while everything
  said "enforcing" (fix: exec `mariadbd` directly). Redis needed the local module
  `homelab_redis_nnp` (`init_nnp_daemon_domain`). Many sandbox options
  (`SystemCallFilter`, `MemoryDenyWriteExecute`, `RestrictNamespaces`, …)
  **imply** NNP – leaving `NoNewPrivileges` out does not help.
- **After every unit/sandbox change: `ps -eZ | grep <daemon>`**, not only
  `systemctl status`. A daemon can be "active (running)" in the wrong domain.
- **K3s only confines pods with `selinux: true` and `k3s-selinux`**, and the
  binary must be relabelled (`restorecon`) after every download.
- **Files Ansible creates inherit the parent directory's type** (e.g. unit
  drop-in directories get `systemd_unit_file_t` instead of `mysqld_unit_file_t`)
  – `restorecon` after creating them.
- **Own domains (`homelab_*`) go through a permissive learning phase first.**
  Export the AVCs, review them, write the rules by hand with refpolicy macros.
  Never apply `audit2allow -M` blindly. Restart the service once in permissive
  mode to catch start-only accesses (bind, config reads) – `audit.log` rotates
  fast while a permissive domain floods it.
- `.fc` paths must use `/var/run/…`, not `/run/…` (`file_contexts.subs_dist`) – on EL9; reversed on EL10, see below.
- **`dontaudit` rules hide denials** – also in the permissive learning phase,
  which then allows them silently. node_exporter could not read
  `/proc/1/mountinfo` after the switch to enforcing (no AVC, just
  `permission denied` in its own log) and the disk metrics were gone for two
  days. After enforcing: check the service's own log and
  `node_scrape_collector_success`; `semodule -DB` shows hidden denials
  (`semodule -B` restores the rules).

### systemd and SSH
- **SSH sessions are sandboxed** (no writes to kernel tunables, no namespaces):
  `sysctl -w`, `nsenter`, `tcpdump` fail there. Use
  `systemd-run --wait --pipe --collect …`.
- **SSH sessions live in sshd's own mount namespace.** A mount made there
  (`ansible.posix.mount state: mounted`) never reaches the host, and a later
  host mount stacks on top of it – the module then misreads the source and
  remounts on every run, doubling the stack (found 2026-11-21: 129 layers of
  `/var/tmp`, host `/tmp` not hardened at all). Mount via `systemd-run`, write
  fstab with `state: present`; read mounts from `/proc/1/mountinfo`.
- **`/tmp` is on disk and `noexec`** on the hosts; it is not emptied by a
  reboot. Clean up downloads/extractions yourself; run scripts from stdin.
- **`PrivateTmp=yes` and `TemporaryFileSystem=/tmp` must not be combined** –
  the private disk directory wins and the tmpfs is silently ignored.

### nftables and fail2ban
- **Replace only our own table** (`table inet filter` / `delete table inet
  filter` / rebuild, one transaction). `flush ruleset` also deletes K3s's tables
  (kube-proxy, flannel, network policies) and breaks pod → service traffic for
  30–90 s.
- **A reload empties our sets** – the handler restarts fail2ban, which re-applies
  its active bans.
- **One set pair per fail2ban jail** (`fail2ban_nft_jails`). With one shared set,
  the unban of a short jail removed addresses that a longer jail still banned.
  Monit's `f2b_sets` check compares fail2ban with the sets.
- **NodePorts are reachable on every local address**; the drop chain runs at
  priority −150 (after conntrack) and drops only `ct state new` – at `raw`
  priority it also dropped replies to outgoing pod connections.
- **Pod network namespaces copy the host's `all`/`default` sysctls** (e.g.
  `log_martians`) – set per interface where only the host is meant.
- MLD packets sit behind a hop-by-hop header, so `nexthdr ipv6-icmp` never
  matches them.

### Kubernetes / K3s
- **`subPath` mounts never update in a running pod.** The pod template carries a
  checksum annotation over every mounted ConfigMap **and Secret**; otherwise a
  changed config or rotated password only arrives at the next accidental restart.
- **NetworkPolicies are enforced by K3s's own controller** (kube-router). Calico
  never matched a pod here (Flannel creates `veth*`, Felix only handles `cali*`).
- **HTTP/3 and reloads:** a worker kept alive by a long-lived connection stays in
  "shutting down" and still holds the shared UDP 443 sockets; about 1 of 8 QUIC
  connections then hung on an 8-worker host (`ps -eo args | grep 'shutting down'`).
  HTTP/2 is not affected. Upstream fixes it only with `quic_bpf`, so HTTP/3 stays off
  on all hosts (the switch `ingress_http3_enabled` remains). Test with several
  client stacks and many runs: 8 clean downloads proved nothing, and my first suspect (the client) was wrong.
- In PromQL, `sum` over a container name adds old and new pods during a rollout
  – aggregate `by (pod)`.

### Nextcloud
- **OPcache runs with `validate_timestamps=0`** – after `occ upgrade` or
  `occ app:update` the pod must be restarted, or old bytecode keeps running.
- Config files load as `config.php`, then `*.config.php` alphabetically (later
  wins). notify_push reads only `config.php`.
- **Previews are tracked in the database** (`oc_previews`) since NC 32 – remove
  them with `occ preview:cleanup`, never with `rm`.
- The preview JPEG quality is an app config value (`preview jpeg_quality`),
  not a `config.php` key.

### Monitoring and alerts
- **Grafana alert rules need `condition`**; with `execErrState: OK` a broken rule
  looks green forever. Check the rule state, not the absence of mail.
- Monit watches the services Prometheus/Grafana cannot alert about themselves
  (if Prometheus stops, every rule stays quietly green).

### AlmaLinux 10 (tested 2026-11-21, production stays on 9)
- **Every 9/10 difference goes through `inventory/group_vars/all/os.yml`** (derived from the facts), never a
  copied role. EL9 must render byte-identical: diff a new template's EL9 output against the old file; explain in
  `.te.j2` with a Jinja comment `{# #}` so the built module on EL9 stays the same and is not rebuilt.
- **SELinux `/run` alias is reversed**: `file_contexts.subs_dist` has `/run /var/run` on EL9 but `/var/run /run`
  on EL10 – `.fc` entries take `os_selinux_run_dir`. The rule above (".fc must use /var/run") is EL9 only.
- **K3s netfilter modules** (`br_netfilter`, `nft_compat`, `xt_*`) are in `kernel-modules-extra` on EL10;
  without them pods cannot reach the API ClusterIP (CoreDNS never ready, Helm timeouts). Check modules
  through `systemd-run`: `ProtectKernelModules` hides `/usr/lib/modules` in SSH sessions.
- No DNF modules, no Redis (Valkey, same `redis_t` types), no `/etc/systemd/journald.conf`, audit rules in
  the `audit-rules` package, k3s-selinux only in Rancher's testing channel (`rpm-testing.rancher.io`).
- **EL10 runs MariaDB 11.8** (`mariadb11.8-server`, Nextcloud 35's recommendation; 10.11 is its minimum).
  11.8 calls `chown` – `~@privileged` killed it with SIGSYS (`status=31/SYS`, audit `SECCOMP syscall=92`).
  **For `SystemCallFilter` the later line wins**: an allow (`@chown`) must sit in a drop-in that sorts
  after the one with `~@privileged` (`99-zz-el10.conf`). A package swap left `mariadb.service` disabled.
- **`restorecon -F` resets the SELinux user too**: files written from an SSH session are `unconfined_u`, so
  `-F` "fixed" them on every run and restarted services for nothing. Use `-R` without `-F`.
- **`ssh` in a script swallows stdin** – a later `read` gets EOF and `set -e` ends the script silently.
  Give such ssh calls `</dev/null`.

### CI
- **No apostrophe in a comment inside a `shell:` block** – ansible-lint fails
  with `parser-error: failed at splitting arguments` (unbalanced quote).
- A Lint failure after ~30 s is usually `ansible-galaxy` (cache/download), not
  the code – re-run before changing anything.
