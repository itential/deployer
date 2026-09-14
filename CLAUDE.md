# itential.deployer — Ansible Collection

## Overview

`itential.deployer` is an Ansible collection (namespace: `itential`, version: `3.7.2`) that deploys the full Itential automation platform stack: Itential Platform (IAP), Itential Automation Gateway (IAG), MongoDB, and Redis (or Valkey as a Remi-free, EL9-only alternative). It supports online and offline (air-gapped) installations, TLS configuration, and multiple deployment topologies.

## Collection Metadata

| Field | Value |
|-------|-------|
| Namespace | `itential` |
| Name | `deployer` |
| Version | `4.0.0` |
| Min ansible-core | `>=2.11, <2.17` |
| Min Python | `>=3.9` |

## Dependencies

| Collection | Version Constraint |
|------------|-------------------|
| `ansible.posix` | `>=0.0.1` |
| `community.general` | `<12.0.0` |
| `community.mongodb` | `>=0.0.1` |

Also requires the `jmespath` Python module on the control node.

## Playbook Inventory

| Playbook | FQCN | Description |
|----------|------|-------------|
| `site.yml` | `itential.deployer.site` | Full stack install: Redis + MongoDB + Platform + Gateway |
| `platform_site.yml` | `itential.deployer.platform_site` | Platform stack only: Redis + MongoDB + Platform |
| `install.yml` | `itential.deployer.install` | Alias for `site.yml` |
| `platform.yml` | `itential.deployer.platform` | Install Itential Platform on `platform`/`platform_secondary` hosts |
| `gateway.yml` | `itential.deployer.gateway` | Install IAG on `gateway` hosts |
| `mongodb.yml` | `itential.deployer.mongodb` | Install MongoDB on `mongodb_primary`, `mongodb_replica`, `mongodb_arbiter` hosts |
| `redis.yml` | `itential.deployer.redis` | Install Redis on `redis_master`/`redis_replica`; Sentinel on `redis_sentinel` hosts |
| `valkey.yml` | `itential.deployer.valkey` | Install Valkey on `valkey_master`/`valkey_replica`; Sentinel on `valkey_sentinel` hosts (EL9 only) |
| `os.yml` | `itential.deployer.os` | Install base OS packages on all component hosts |
| `nginx.yml` | `itential.deployer.nginx` | Install and configure nginx (wraps `nginxinc.nginx` and `nginxinc.nginx_config`) |
| `nginx_install.yml` | `itential.deployer.nginx_install` | Install nginx only |
| `nginx_configure.yml` | `itential.deployer.nginx_configure` | Configure nginx, SELinux booleans, firewalld port, restart |
| `patch_platform.yml` | `itential.deployer.patch_platform` | Upgrade Itential Platform in-place |
| `patch_gateway.yml` | `itential.deployer.patch_gateway` | Upgrade IAG in-place |
| `certify.yml` | `itential.deployer.certify` | Run all certification playbooks (Redis + MongoDB + Platform) |
| `certify_redis.yml` | `itential.deployer.certify_redis` | Generate Redis/Sentinel installation certification reports |
| `certify_valkey.yml` | `itential.deployer.certify_valkey` | Generate Valkey/Sentinel installation certification reports |
| `certify_mongodb.yml` | `itential.deployer.certify_mongodb` | Generate MongoDB installation certification reports |
| `certify_platform.yml` | `itential.deployer.certify_platform` | Generate Platform installation certification reports |
| `verify.yml` | `itential.deployer.verify` | Pre-install environment verification (OS, HW specs, proxy, required repository connectivity) for all components |
| `verify_redis.yml` | `itential.deployer.verify_redis` | Pre-install verification for Redis hosts |
| `verify_valkey.yml` | `itential.deployer.verify_valkey` | Pre-install verification for Valkey hosts |
| `verify_mongodb.yml` | `itential.deployer.verify_mongodb` | Pre-install verification for MongoDB hosts |
| `verify_platform.yml` | `itential.deployer.verify_platform` | Pre-install verification for Platform hosts |
| `verify_gateway.yml` | `itential.deployer.verify_gateway` | Pre-install verification for Gateway hosts |
| `download_packages_site.yml` | `itential.deployer.download_packages_site` | Download all packages for offline install (Platform stack + Gateway) |
| `download_packages_platform_site.yml` | `itential.deployer.download_packages_platform_site` | Download all packages for Platform stack offline install |
| `download_packages_platform.yml` | `itential.deployer.download_packages_platform` | Download Platform packages for offline install |
| `download_packages_gateway.yml` | `itential.deployer.download_packages_gateway` | Download Gateway packages for offline install |
| `download_packages_gateway_site.yml` | `itential.deployer.download_packages_gateway_site` | Wraps `download_packages_gateway` with a tag |
| `download_packages_mongodb.yml` | `itential.deployer.download_packages_mongodb` | Download MongoDB packages for offline install |
| `download_packages_redis.yml` | `itential.deployer.download_packages_redis` | Download Redis packages for offline install |
| `download_packages_valkey.yml` | `itential.deployer.download_packages_valkey` | Download Valkey packages for offline install |
| `download_packages_os.yml` | `itential.deployer.download_packages_os` | Download OS packages for offline install |

## Inventory Topology Options

| Topology | Directory | Description |
|----------|-----------|-------------|
| `aio` | `example_inventories/aio/` | All-in-one: all components on a single host. For dev/CI use. TLS off by default. |
| `minimal` | `example_inventories/minimal/` | Single instance of each component on separate hosts. Exercises network paths. TLS configurable. |
| `ha2` | `example_inventories/ha2/` | High-availability: 2 Platform nodes, 3-node MongoDB replica set, 3-node Redis+Sentinel cluster, 1 Gateway. Recommended for test and production. |
| `asa` | `example_inventories/asa/` | Active/Standby: 5-node MongoDB (4 data + 1 arbiter across 3 DCs), 4-node Redis across 3 DCs. Disaster recovery topology. |
| `platform` | `example_inventories/platform/` | Platform-only example showing external (managed) Redis/MongoDB via URL. |
| `redis` | `example_inventories/redis/` | Redis-only examples: install from Remi repo, from system repo, or from source. |
| `valkey` | `example_inventories/valkey/` | Valkey-only example: install via the native EL9 AppStream package (the only supported method). |

## Roles Summary

| Role | Purpose |
|------|---------|
| `common` | Shared defaults (`offline_install_enabled`, `common_install_yum_repos`, release file path). No tasks in main. |
| `os` | Installs base OS and security packages (RedHat family only). Skips if `itential-release` file exists. |
| `python` | Shared utility role for installing Python packages and pip dependencies (no defaults of its own; callers pass vars). |
| `selinux` | Shared utility role for compiling and installing custom SELinux `.te` policy modules from the calling role's `files/` directory. |
| `offline` | Shared utility role for downloading and installing RPMs/wheels/adapters in air-gapped mode. |
| `mongodb` | Installs and configures MongoDB: users, replica set, auth, TLS, kernel tuning, SELinux, logrotate, NUMA. |
| `redis` | Installs and configures Redis (from source or repo) and Redis Sentinel: auth, TLS, replication, SELinux. |
| `valkey` | Installs and configures Valkey (via the native EL9 AppStream package only) and Valkey Sentinel: auth, TLS, replication. No source install, no Remi, no EL8 support, no role-level SELinux step (handled entirely by the base OS policy). |
| `platform` | Installs and configures Itential Platform: NodeJS, Python, RPM packages, adapters, properties file, TLS certs, Vault, SELinux. |
| `gateway` | Installs and configures Itential Automation Gateway (IAG): Python venv, Ansible, Nornir, TLS certs, systemd service, SELinux. |

## Key Global Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `platform_release` | (required) | Platform release version (e.g., `6`). Drives package/Python/NodeJS selection. |
| `offline_install_enabled` | `false` | Enables air-gapped installation mode across all roles. |
| `offline_target_node_root` | `/var/tmp` | Root directory on target nodes for offline packages. |
| `offline_control_node_root` | `{{ playbook_dir }}/files` | Root directory on the control node where offline packages are staged. |
| `offline_itential_packages_path` | `itential_packages/{{ distro }}_{{ major_ver }}` | Subdirectory within offline roots for per-OS package sets. |
| `common_install_yum_repos` | `true` | When `false`, skips YUM repo installation (use internal repos instead). |
| `common_itential_release_file` | `/etc/itential-release` | Path to the file tracking installed component versions. |
| `env` | (required for verify) | Deployment environment (`dev`, `test`, `prod`). Used for hardware spec validation. |
| `repository_username` | (optional) | Username for downloading packages from Nexus/JFrog. |
| `repository_password` | (optional) | Password for downloading packages from Nexus/JFrog. |
| `repository_api_key` | (optional) | API key for downloading packages from JFrog. |

## TLS Overview

TLS is supported at the component level and is **enabled by default** for MongoDB, Platform, and Valkey, and **disabled by default** for Redis.

| Component | Enable Flag | Copy-Certs Flag | PKI Base Dir |
|-----------|------------|-----------------|--------------|
| MongoDB | `mongodb_tls_enabled: true` | `mongodb_tls_copy_certs: true` | `/etc/pki/mongodb` |
| Redis | `redis_tls_enabled: false` | n/a (always copies when TLS enabled) | `/etc/pki/redis` |
| Valkey | `valkey_tls_enabled: true` | n/a (always copies when TLS enabled) | `/etc/pki/valkey` |
| Platform (HTTPS) | `platform_webserver_https_enabled: true` | `platform_webserver_https_copy_certs: true` | `/etc/pki/itential-platform/https` |
| Platform (MongoDB client) | `platform_mongo_tls_enabled: true` | `platform_mongodb_copy_certs: true` | `/etc/pki/itential-platform/mongodb` |
| Gateway (HTTPS) | `gateway_https_enabled: true` | `gateway_pki_copy_certs: true` | `/etc/pki/automation-gateway` |

The deployer does **not** generate certificates. Certificates must be provided on the Ansible control node; the deployer copies them to targets. Each role uses a `*_pki_src_dir` variable pointing to the certificate source directory on the control node.

## Offline Install

When `offline_install_enabled: true`, the deployer skips all internet-facing package downloads and instead installs from a pre-staged local directory. The `download_packages_*` playbooks are used to stage packages while the target still has internet access (or from a machine that does). The offline directory layout is:

```
files/
  itential_packages/
    <distro>_<major_ver>/
      <platform_release>/
        platform/rpms/
        platform/wheels/
        platform/adapters/
      <gateway_release>/
        gateway/rpms/
        gateway/wheels/
        gateway/collections/
```

See `docs/offline_install_guide.md` for the full workflow.

## Custom Plugins / Modules

| Module | Purpose |
|--------|---------|
| `itential.deployer.gather_host_information` | Collects OS, architecture, and hardware facts used by the `verify-host.yml` task file. |
| `itential.deployer.mongodb_config_state` | Queries a running MongoDB instance to determine whether auth and replication are already configured. Used to make user-creation tasks idempotent. |

## Docs Index

| File | Contents |
|------|----------|
| `docs/itential_platform_guide.md` | Platform role variables, adapters, Vault config, logging, webserver settings |
| `docs/itential_gateway_guide.md` | Gateway role variables, feature flags, Ansible/Nornir config |
| `docs/mongodb_guide.md` | MongoDB role variables, replica set, TLS, user accounts |
| `docs/redis_guide.md` | Redis role variables, Sentinel, TLS, install methods |
| `docs/valkey_guide.md` | Valkey role variables, Sentinel, TLS, EL9-only AppStream install |
| `docs/tls_guide.md` | End-to-end TLS configuration guide across all components |
| `docs/offline_install_guide.md` | Step-by-step offline (air-gapped) install workflow |
| `docs/patch_itential_platform_guide.md` | How to run `patch_platform.yml` to upgrade Platform |
| `docs/patch_itential_gateway_guide.md` | How to run `patch_gateway.yml` to upgrade IAG |
| `docs/preflight_guide.md` | How to run the `verify` playbooks and interpret results |
| `docs/certify_guide.md` | How to run the `certify` playbooks and interpret the certification report |
| `docs/nginx.md` | Nginx reverse-proxy setup and configuration |
| `docs/release_notes_v4.md` | Release notes for v4 changes |

## CI

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ansible-lint.yml` | push/PR to `main`, `dev` | Lint validation |
| `role-readme-check.yml` | push/PR to `dev` | Fails the build if any `roles/*/` directory is missing a README (Ansible Galaxy requirement) |
| `updateChangelog.yml` | GitHub release or manual | Updates the changelog |
| `publish_ansible_collection.yml` | GitHub release or manual | Publish to Galaxy |
