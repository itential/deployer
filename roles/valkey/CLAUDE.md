# Role: valkey

## Purpose

Installs and configures Valkey and Valkey Sentinel for use with Itential Platform, as a
Remi-free, non-source-compiled alternative to `roles/redis`. Handles authentication (ACL
users), TLS, replication, Sentinel setup, and firewalld. Unlike `roles/redis`, this role has
**no source-install path and no Remi dependency** — Valkey is installed exclusively via the
native OS package repositories (the AppStream module on EL9, Amazon Linux 2023's own core
repo).

## Supported Platforms

**RHEL/Rocky/AlmaLinux 9, and Amazon Linux 2023.** There is no supported install path for EL8
(no AppStream module stream exists for Valkey on EL8, and this role deliberately does not
compile from source to reach it — see "Design Decisions" below). A host in `valkey_master` /
`valkey_replica` / `valkey_sentinel` on any other OS or major version fails
`validate-vars.yml` immediately with an explicit error. Use `roles/redis` on EL8.

Install paths are also not customizable on either platform — the Valkey RPM is not
relocatable, unlike a from-source install. Customers requiring non-standard install locations
must use `roles/redis` instead.

## Entry Point Tasks — main.yml

1. Validate variables (`validate-vars.yml`) and set node type facts:
   `valkey_is_master_node`, `valkey_is_replica_node`, `valkey_is_sentinel_node`,
   `valkey_is_data_node`, `valkey_has_replicas`, `valkey_has_sentinels`.
2. Install Valkey (triggers `Enable and Start Valkey` and `Enable and Start Valkey Sentinel`
   handlers):
   a. `install-common.yml` — create OS user/group
   b. `install-from-repo.yml` — `dnf install valkey` (or offline RPM install)
3. Configure TLS (`configure-valkey-tls.yml` / `configure-sentinel-tls.yml`) — when
   `valkey_tls_enabled: true`
4. Configure Valkey (`configure-valkey.yml`) — when `valkey_is_data_node`
5. Configure Sentinel (`configure-sentinel.yml`) — when `valkey_is_sentinel_node`
6. Flush handlers, start and assert Valkey is active (data nodes)
7. Flush handlers, start and assert Valkey Sentinel is active (sentinel nodes)

There is no SELinux configuration step in this role at all (see "Design Decisions").

## Key Variables

### valkey.yml defaults

| Variable | Default | Purpose |
|----------|---------|---------|
| `valkey_bin_dir` | `/usr/bin` | Binary location (package install only, not overridable — see validate-vars.yml) |
| `valkey_conf_dir` | `/etc/valkey` | Config directory |
| `valkey_log_dir` | `/var/log/valkey` | Log directory |
| `valkey_data_dir` | `/var/lib/valkey` | Data directory |
| `valkey_port` | `6379` | Listen port |
| `valkey_auth_enabled` | `true` | Enable ACL user authentication |
| `valkey_tls_enabled` | `true` | Enable TLS |
| `valkey_replicaof` | `{{ groups['valkey_master'][0] }} {{ valkey_port }}` | Replication target (replica nodes) |
| `valkey_replica_priority` | `auto` | Sentinel failover priority |
| `valkey_user_admin_password` / `valkey_user_itential_password` / etc. | see file | ACL user passwords |
| `valkey_maxmemory_bytes` | `auto` | `auto` = ratio of RAM; or explicit bytes |

### sentinel.yml, pki.yml, offline.yml defaults

Same shape as the equivalent `roles/redis` files, `valkey_` prefixed.

### install.yml defaults / vars/platform-release-6.yml

There is no `valkey_install_from_source` variable — installation is always via package. The
only EL major version keys present in `valkey_packages_default` are `"9"` and `"2023"` (both
value `["valkey"]`, the package name on both platforms). Looking this dict up for any other
major version does not happen because `validate-vars.yml` already asserts one of those two
first.

### vars/main.yml

`valkey_required_repositories` has a **single entry**: the Rocky/AlmaLinux AppStream mirror
(`mirrors.rockylinux.org`). Unlike `roles/redis` there is no third-party repository (Remi,
EPEL, upstream source) to check. This one entry is a best-effort connectivity check — it does
not distinguish RHEL (which resolves AppStream through its own subscription-manager CDN, not a
fixed public URL) or Amazon Linux 2023 (which ships Valkey in its own preconfigured core repo)
from Rocky/Alma; on those platforms the check still runs but isn't actually verifying the repo
that host will use.

## Design Decisions

These are deliberate departures from a 1:1 mirror of `roles/redis`, decided during scoping:

- **No source install, ever.** `roles/redis` defaults to compiling from source
  (`redis_install_from_source: true`). This role never does — `install-from-source.yml`,
  `install-remi-repo.yml`, `valkey_build_packages`, and the Remi/EPEL repo URL vars do not
  exist in this role at all.
- **No Remi.** Package installs always come from the native OS package repositories, never Remi.
- **EL8 is explicitly unsupported**, not silently skipped. No EPEL8 fallback (even though
  EPEL8 does carry a Valkey package) and no source-compile fallback. A host on EL8 fails hard
  in `validate-vars.yml`.
- **Amazon Linux 2023 is supported alongside EL9** — confirmed AL2023 ships `valkey` natively
  via its own core repo (no module-stream concept like RHEL's AppStream). Both major version
  keys (`"9"`, `"2023"`) map to the same `["valkey"]` package list in
  `vars/platform-release-6.yml`.
- **Not relocatable.** The Valkey RPM has no `Prefix:` tag, so unlike `roles/redis`'s
  from-source path there is no way to install to a non-default location on either supported
  platform. `validate-vars.yml`'s directory-override assert (see above) is a hard requirement,
  not just this role's own preference.
- **Version is whatever the OS's own package repository currently resolves to** (8.0.7 on EL9
  AppStream as of this writing) — not a specific upstream tag pinned via source tarball, unlike
  `redis_source_url`.
- **No role-level SELinux configuration.** Confirmed via the upstream
  `fedora-selinux/selinux-policy` `redis.fc` file: it already equivalences standard Valkey
  paths (`/usr/bin/valkey-server`, `/etc/valkey`, `/var/lib/valkey`, `/var/log/valkey`,
  `/run/valkey`) onto the same `redis_exec_t`/`redis_conf_t`/`redis_log_t`/`redis_var_lib_t`/
  `redis_var_run_t` types Redis uses. Since this role only ever installs to standard paths via
  the AppStream package, SELinux labeling is handled entirely by the base OS policy with zero
  extra role tasks — no equivalent of `roles/redis/tasks/configure-selinux.yml` or
  `itential_redis_sentinel.te` exists here. If a future change introduces a non-default
  `valkey_bin_dir` or a source-install path, this decision needs revisiting.
- **No `install-common.yml` directory-creation tasks.** The AppStream package's own `%files`
  already creates `/etc/valkey`, `/var/lib/valkey`, `/var/log/valkey` with correct ownership
  and SELinux context — this role does not recreate them (this mirrors how `roles/redis`
  already skips its own directory-creation tasks when installing from a package instead of
  source; here that's the *only* path, so the tasks were removed rather than left as
  permanently-`false` conditionals).

## Templates

Same set as `roles/redis`, renamed (`valkey.conf.j2`, `sentinel.conf.j2`, `valkey.service.j2`,
`valkey-sentinel.service.j2`, `valkey.logrotate.j2`, `valkey-sentinel.logrotate.j2`,
`valkey-validation-report.md.j2`). Config directive syntax is unchanged from Redis's — Valkey
is protocol- and config-compatible — only comments and variable names were updated.

## Inventory Groups

| Group | Role |
|-------|------|
| `valkey_master` | Primary data node (EL9 or Amazon Linux 2023 only) |
| `valkey_replica` | Secondary data nodes (EL9 or Amazon Linux 2023 only) |
| `valkey_sentinel` | Sentinel nodes (EL9 or Amazon Linux 2023 only) |

## Not Yet Done

- Root playbooks (`valkey.yml`, `verify_valkey.yml`, `certify_valkey.yml`,
  `download_packages_valkey.yml`) and their unconditional imports into `platform_site.yml` /
  `verify.yml` / `certify.yml` / `download_packages_platform_site.yml`.
- New example inventories under `example_inventories/valkey/`.
- `docs/valkey_guide.md`.
- End-to-end verification in the PE lab (package install, TLS, Sentinel failover, SELinux
  enforcing mode, Platform connectivity) — nothing in this role has been run against a real
  host yet.
