# Valkey Role

The playbook and role in this section install and configure Valkey for the Itential Platform,
as a Remi-free, non-source-compiled alternative to the `redis` role. There is one
Valkey-related role which installs Valkey and performs a base configuration. Optionally
configures authentication, TLS, and replication.

## What is Valkey?

Valkey is a BSD-licensed, community-governed fork of Redis, created after Redis changed its
license away from a fully open-source model. Valkey is protocol- and config-compatible with
Redis: it speaks the same RESP protocol, uses the same `redis.conf`-style configuration
grammar, and implements the same Sentinel protocol and ACL directives. This is why the
`valkey` role and its configuration files look nearly identical to the `redis` role's -- under
the hood, Valkey behaves the same way Redis does for everything this deployer configures
(authentication, TLS, replication, Sentinel failover). The Itential Platform connects to
either backend the same way, since Platform's client only speaks RESP and does not care which
server product is on the other end.

## Supported Platforms

**RHEL/Rocky/AlmaLinux 9 only.** Unlike the `redis` role, there is no supported install path
for EL8:

- EL8's AppStream does not ship a Valkey module stream at all.
- EPEL8 does carry a Valkey package, but this deployer standardizes on the native AppStream
  module rather than mixing package sources, so EPEL8 is not used.
- Compiling Valkey from source would work on EL8, but this deployer deliberately does not
  support source installs for Valkey at all (unlike `redis_install_from_source`, there is no
  equivalent `valkey_install_from_source` flag or source-install code path in this role).

A host placed in `valkey_master`, `valkey_replica`, or `valkey_sentinel` on EL8 (or any
non-RedHat-family OS, or EL9) fails `validate-vars.yml` immediately with an explicit error
telling the operator to use the `redis` role instead.

## Valkey Install

The `valkey` role performs a base install of Valkey from the native EL9 AppStream package
(`dnf install valkey`) including any OS packages required. It creates the appropriate Linux
users, log files, and systemd services. It uses a template to generate a configuration file
based on the variables defined in the valkey group vars. It will start the Valkey service when
complete.

Unlike `redis`, this role does not create `/etc/valkey`, `/var/lib/valkey`, or
`/var/log/valkey` itself, and does not install or manage any SELinux profiles -- the AppStream
package's own `%files` already creates those directories with the correct ownership, and the
base OS's `selinux-policy` package already maps standard Valkey paths onto the same
`redis_exec_t`/`redis_conf_t`/`redis_log_t`/`redis_var_lib_t` SELinux types Redis uses, so no
extra role-level SELinux work is required as long as the default paths are used (which they
always are -- overriding `valkey_bin_dir`, `valkey_conf_dir`, `valkey_data_dir`, or
`valkey_log_dir` is rejected by `validate-vars.yml`).

## Authentication

Optionally, the `valkey` role performs tasks to require authentication (username and password)
when communicating with the Valkey server. It adjusts the Valkey config file and adds each of
the required users and applies appropriate ACLs (see table). The "default" Valkey user is
disabled. It modifies the Valkey config file to use the appropriate user while doing
replication. It adjusts the Sentinel config file to enable the correct Sentinel user to
monitor the Valkey cluster, if required. It disables the default user in both Valkey and
Valkey Sentinel.

| User Name | Default Password | Description |
| :-------- | :--------------- | :---------- |
| admin | admin | Has full access to the Valkey database. |
| itential | itential | Has access to all keys, all channels, and all commands except: -asking -cluster -readonly -readwrite -bgrewriteaof -bgsave -failover -flushall -flushdb -psync -replconf -replicaof -save -shutdown -sync |
| repluser | repluser | Has access to the minimum set of commands to perform replication. |
| sentineluser | sentineluser | Has access to the minimum set of commands to perform sentinel monitoring. |
| monitor | monitor | Has access to the minimum set of commands to gather metric and cluster data from Valkey and Sentinel. |

:::(Warning) (⚠ Warning: ) It is assumed that these default passwords will be changed to meet
more rigorous standards. These are intended to be defaults strictly used just for ease of the
installation. It is highly recommended that sensitive data be encrypted using Ansible Vault.

## TLS

TLS is **enabled by default** for Valkey and Valkey Sentinel. Both are controlled by a single
flag (`valkey_tls_enabled`). When TLS is enabled, the deployer enforces TLSv1.3 only and
disables plain-text connections by setting `port 0` and listening exclusively on `tls-port`.

Client certificate authentication is disabled by default (`valkey_tls_auth_clients: no`). The
connection is still fully encrypted; the server just does not require clients to present a
certificate.

### Required Certificates

The deployer does not generate certificates. The following files must be present on the
Ansible control node in the directory defined by `valkey_pki_src_dir`:

| File | Variable | Description |
| :--- | :------- | :---------- |
| `<hostname>.crt` | `valkey_tls_cert_file` | Server certificate (one per node by default) |
| `<hostname>.key` | `valkey_tls_key_file` | Server private key (one per node by default) |
| `ca-bundle.crt` | `valkey_tls_ca_file` | CA bundle used to verify peer certificates |
| `<hostname>.crt` | `valkey_sentinel_tls_cert_file` | Sentinel certificate (one per Sentinel node by default, named after inventory hostname) |
| `<hostname>.key` | `valkey_sentinel_tls_key_file` | Sentinel private key (one per Sentinel node by default) |

### Example Inventory - TLS Enabled (Single Node)

```yaml
all:
  children:
    valkey_master:
      hosts:
        <host1>:
          ansible_host: <addr1>
      vars:
        platform_release: 6
        valkey_pki_src_dir: /path/to/certs/on/control/node
```

### Example Inventory - TLS Enabled (Sentinel HA)

```yaml
all:
  vars:
    platform_release: 6
    valkey_pki_src_dir: /path/to/certs/on/control/node
  children:
    valkey_master:
      hosts:
        <host1>:
          ansible_host: <addr1>
    valkey_replica:
      hosts:
        <host2>:
          ansible_host: <addr2>
        <host3>:
          ansible_host: <addr3>
    valkey_sentinel:
      hosts:
        <host1>:
          ansible_host: <addr1>
        <host2>:
          ansible_host: <addr2>
        <host3>:
          ansible_host: <addr3>
```

## Replication

Optionally, the `valkey` role performs the steps required to create a Valkey replica set. It
uses a template to generate a Valkey Sentinel config file. It modifies the Valkey config file
to turn off protected-mode. It assumes that the first host defined in the inventory file is
the initial primary. It will update the config file for the non-primary Valkey servers to
replicate from the primary using hostname. It will start the Valkey Sentinel service when
complete. Replication and Sentinel failover mechanics are unchanged from Redis -- see the
`redis_guide.md` Replication section for the full explanation of replica priority and quorum
calculation, which apply identically here (`valkey_replica_priority`, `valkey_sentinel_quorum`).

## Automatic Valkey Maxmemory Calculation

When `valkey_maxmemory_bytes` is set to `auto`, the installation process automatically
calculates the Valkey `maxmemory` value based on the system's total available RAM, using the
same formula as the `redis` role:

`maxmemory = max(valkey_maxmemory_min_mb, system_ram × valkey_maxmemory_ratio)`

## Variables

### Global Variables

The variables in this section can be configured in the inventory in the `all` group or the
`valkey` group.

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `platform_release` | Fixed-point | Designates the Itential Platform major version. | N/A |

Defining `platform_release` in the inventory is optional. However, this variable is used to
determine the default `valkey_packages` value. If `platform_release` is not defined, then
`valkey_packages` must be defined.

### Valkey Variables

The following table lists the default variables located in `roles/valkey/defaults/main/valkey.yml`.

| Variable | Type | Description | Default Value |
| :------- | :--- | :---------- | :------------ |
| `valkey_bin_dir` | String | The Valkey binary directory. Not overridable -- see `validate-vars.yml`. | `/usr/bin` |
| `valkey_conf_dir` | String | The Valkey configuration directory. Not overridable. | `/etc/valkey` |
| `valkey_conf_file` | String | The location of the Valkey configuration file. | `/etc/valkey/valkey.conf` |
| `valkey_log_dir` | String | The Valkey log directory. Not overridable. | `/var/log/valkey` |
| `valkey_data_dir` | String | The location of the Valkey data directory. Not overridable. | `/var/lib/valkey` |
| `valkey_port` | Integer | The Valkey listen port. | `6379` |
| `valkey_owner` / `valkey_group` | String | The Valkey Linux user/group. | `valkey` |
| `valkey_tls_enabled` | Boolean | Flag to enable TLS connections. Also enables Sentinel TLS when Sentinel is in use. | `true` |
| `valkey_maxmemory_bytes` | String/Integer | Maximum memory Valkey can use. See Automatic Maxmemory Calculation above. | `auto` |
| `valkey_certify_report_dir_remote` / `_local` | String | Directories for certification reports. | `/var/tmp/itential-reports/valkey`, `/tmp/itential-reports/valkey` |

### Auth, Replication, Sentinel, and PKI Variables

These follow the exact same shape as the `redis` role's equivalents, `valkey_`-prefixed
instead of `redis_`-prefixed (e.g. `valkey_user_admin_password`, `valkey_replicaof`,
`valkey_sentinel_quorum`, `valkey_pki_base_dir`). See `redis_guide.md`'s corresponding tables
for full descriptions -- the semantics are identical.

### Offline Variables

There are several variables used when downloading and installing Valkey in offline mode.
These variables will not be documented here since they will rarely need to be overridden in
the inventory.

## Installation Method

Unlike `redis`, this role does not support installing from source or from Remi -- there is no
`valkey_install_from_source` flag. Valkey is always installed via `dnf` using the package(s)
in `valkey_packages`, which defaults to the native EL9 AppStream `valkey` package when
`platform_release` is defined. The current default values can be found in
`roles/valkey/vars/platform-release-<platform_release>.yml`, which only defines an entry for
EL major version `9`.

## Building Your Inventory

To install and configure Valkey, add a `valkey_master` group and host(s) to your inventory.
The following inventory shows a basic Valkey configuration with a single Valkey node on EL9
with authentication.

### Example Inventory - Single Valkey Node

```yaml
all:
  children:
    valkey_master:
      hosts:
        <host1>:
          ansible_host: <addr1>
    vars:
        platform_release: 6
```

To configure a Valkey replica set, add the replica hosts to the `valkey_replica` group and
configure the `valkey_replicaof` variable.

### Example Inventory - Configure Valkey Replication

```yaml
all:
  vars:
    platform_release: 6
  children:
    valkey_master:
      hosts:
        <host1>:
          ansible_host: <addr1>

    valkey_replica:
      hosts:
        <host2>:
          ansible_host: <addr2>
        <host3>:
          ansible_host: <addr3>
      vars:
        valkey_replicaof: <master-hostname-or-ip> <valkey-port> # defaults to "{{ groups['valkey_master'][0] }} {{ valkey_port}}"
```

To configure Sentinels, add the sentinel hosts to the `valkey_sentinel` group.

```yaml
all:
  vars:
    platform_release: 6
  children:
    valkey_master:
      hosts:
        <host1>:
          ansible_host: <addr1>

    valkey_replica:
      hosts:
        <host2>:
          ansible_host: <addr2>
        <host3>:
          ansible_host: <addr3>
      vars:
        valkey_replicaof: <master-hostname-or-ip> <valkey-port> # defaults to "{{ groups['valkey_master'][0] }} {{ valkey_port}}"

    valkey_sentinel:
      hosts:
        <host4>:
          ansible_host: <addr4>
        <host5>:
          ansible_host: <addr5>
        <host6>:
          ansible_host: <addr6>
```

## Running the Playbook

To execute the Valkey role, run the `valkey` playbook:

```bash
ansible-playbook itential.deployer.valkey -i <inventory>
```
