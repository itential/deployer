# Ansible Collection - itential.deployer

## Table of contents

1. [Overview](#overview)
2. [Supported Architectures](#supported-architectures)
    1. [All-in-one Architecture](#all-in-one-architecture)
    2. [Minimal Architecture](#minimal-architecture)
    3. [Highly Available Architecture](#highly-available-architecture)
    4. [Active/Standby Architecture](#activestandby-architecture)
3. [Deployer Prerequisites](#deployer-prerequisites)
    1. [Required Python, Ansible, and Ansible modules](#required-python-ansible-and-ansible-modules)
    2. [Required Public Repositories](#required-public-repositories)
    3. [Ports and Networking](#ports-and-networking)
    4. [Certificates](#certificates)
    5. [Passwords](#passwords)
    6. [Obtaining the Itential Binaries](#obtaining-the-itential-binaries)
4. [Installing and Upgrading the Deployer](#installing-and-upgrading-the-deployer)
    1. [Online Installation](#online-installation)
    2. [Offline Installation](#offline-installation)
5. [Running the Deployer](#running-the-deployer)
    1. [Confirm Requirements](#confirm-requirements)
    2. [Determine the Working and Deployer Directories](#determine-the-working-and-deployer-directories)
    3. [Create the Inventories Directory](#create-the-inventories-directory)
    4. [Download Installation Artifacts](#download-installation-artifacts)
    5. [Copy Installation Artifacts into the Files Directory](#copy-installation-artifacts-into-the-files-directory)
    6. [Create a Symlink to the Files Directory](#create-a-symlink-to-the-files-directory)
    7. [Create the Inventory File](#create-the-inventory-file)
    8. [Run the Itential Deployer](#run-the-itential-deployer)
    9. [Confirm Successful Installation](#confirm-successful-installation)
6. [Component Guides](#component-guides)
    1. [MongoDB](#mongodb)
    2. [Redis](#redis)
    3. [Valkey](#valkey)
    4. [Itential Platform](#itential-platform)
    5. [Itential Gateway](#itential-gateway)
7. [Patching Itential Platform and IAG](#patching-itential-platform-and-iag)
8. [Using Internal YUM Repositories](#using-internal-yum-repositories)
9. [Running the Deployer in Offline Mode](#running-the-deployer-in-offline-mode)
10. [Appendix A: Definition of "Highly Available" Dependencies](#appendix-a-definition-of-highly-available-dependencies)

## Overview

An Itential environment is composed of several applications working in conjunction with one another.
At its most basic, the following must be installed.

- Itential Platform
- Itential Automation Gateway (IAG)
- Redis (or Valkey as a Remi-free, EL9-only alternative)
- MongoDB

Optionally, one can include Hashicorp Vault for secrets management, and Prometheus & Grafana for
metrics analysis and alerting.

In many environments, these applications are installed across multiple systems to improve resiliency
and performance. To assist users with such installations, Itential provides the **Itential
Deployer**.

The Itential deployer can deploy supported Itential architectures.

## Supported Architectures

- All-in-one Architecture
- Minimal Architecture
- Highly Available Architecture
- Active/Standby Architecture

### All-in-one Architecture

The All-In-One architecture is an architecture where all components are installed on the same
instance.  This architecture lends itself well to development environments and “throw away”
environments like those found in a CI/CD pipeline.  Security considerations are non-existent
or use simple default passwords as the emphasis is placed on simplicity.

### Minimal Architecture

A Minimal Architecture is an architecture where all or most components are single instances and can
not gracefully tolerate failures. This architecture lends itself well to development environments.
It favors the engineer with its simplicity and enables them to do their work with few restrictions.
Security considerations should lean towards openness and ease of use but at the same time capture
the spirit of the other higher environments. In this architecture, each of the required Itential
components is a single instance.  This architecture will exercise the network connectivity between
the components and can be advantageous to deploy as a development environment.

The ideal minimal architecture will have 4 VM's with each hosting a single instance of the required
components. An acceptable variation is to have 1 VM with everything hosted on it. The number of
VM's and what is hosted where is less of a concern with the MA because the intent is simplicity and
to enable engineers to build automations and not be a highly-available environment.

Itential recommends applying security principles to ALL environments. In the MA, this would include
configuring all components to use authentication. Optionally, we recommend using SSL when
communicating with components on other VMs but recognize that this should be enabled at the
discretion of the customer.

### Highly Available Architecture

A Highly Available Architecture is an architecture where all or most components are redundant and
can gracefully tolerate at least 1 catastrophic failure. This architecture is the recommended
architecture for testing environments and simple production environments. The intent is to provide
an environment that is as close to production as possible so that testing automations can provide
confidence and expose potential problems sooner in the development lifecycle. Security
considerations should mimic production requirements. In this architecture, each of the required
Itential components are installed in clusters to provide stability, mimic production, and expose any
issues with the clustering of the components. This could also serve as a production architecture.

The ideal HA2 environment will have 9 VMs:

- 2 VMs hosting Itential Platform.
- 3 VMs hosting MongoDB configured as a replica set.
- 3 VMs hosting Redis (or Valkey, on EL9-only hosts) configured as a highly available replica set using Sentinel.
- 1 VM hosting IAG.

Itential recommends applying sound security principles to ALL environments. This would include
configuring all components to use authentication within the HA2. Additionally, we recommend using
SSL when communicating with components on other VMs and across clusters.

### Active/Standby Architecture

An Active/Standby Architecture (ASA) is an architecture (that is normally used for making HA2
architectures redundant) reserved for production environments where Disaster Recovery is required.
The intent is to provide a secondary environment that can be used in the event of a disaster in the
active datacenter. The secondary stack should be preconfigured to be quickly made into the primary
stack. Care needs to be taken that the same level of access to 3rd party systems existing in the
secondary stack matches those in the primary stack. Security must be taken into account if this is
used as a production environment.

The ASA architecture MUST utilize 3 data centers in order to be fully geographically redundant. This
is due to the way that MongoDB and Redis handle data replication and the promotion of a new primary
in disaster scenarios.

The MongoDB cluster will consist of 5 members: 4 data-bearing members and a MongoDB arbiter, 2
data-bearing members in the primary data center, 2 data-bearing members in the secondary data
center, and the arbiter in a third data center. This will preserve a majority of voting members (3)
in the event of a data center loss.

Redis must also consider the preservation of voting members in this architecture but it works
slightly different than MongoDB. Redis will consist of 4 data-bearing members, 2 data-bearing
members in the primary data center, 2 data-bearing members in the secondary data center. Unlike
MongoDB, Redis HA requires Redis Sentinel. These must be distributed in 3 data centers to preserve
a majority of voting members (3) in the event of a data center loss.

Valkey is a protocol-compatible, drop-in alternative to Redis and follows the identical HA mechanics
described above: same replica/Sentinel topology, same quorum math, same data center distribution.
It is currently only supported on EL9 hosts, installed via the native AppStream package (no Remi, no
source install) — see the [Valkey Guide](docs/valkey_guide.md) for details.

Itential recommends applying sound security principles to ALL environments. In the ASA, this would
include configuring all components to use authentication and use SSL when communicating with
components on other VMs and across clusters.

### Alternative Architectures

Its not unusual to "outsource" the management of the dependencies (Redis and MongoDB) to
either other internal teams or to external vendors such as AWS. Itential provides a way to leverage
these solutions simply by configuring them accordingly and as long as these solutions comply with
the basic requirements. For example, if Elasticache is chosen to fulfill the Redis needs then it
must use the "Redis" version and not "Memcache". Likewise for MongoDB, Mongo Atlas is supported but
not DynamoDB which is not the same. See the examples for how to implement.

## Deployer Prerequisites

The Itential Deployer is an Ansible project and as such requires running on a control node. That
node has its own set of dependencies.

### Control Node Specifications

Itential recommends using a dedicated node running the requirements listed below as the ansible
control node for the deployer project. That node should meet or exceed the following specifications:

| Component | Value                |
|-----------|----------------------|
| OS        | RHEL8/9 or Rocky 8/9 |
| RAM       | 4 GB                 |
| CPUs      | 2                    |
| Disk      | 20 GB                |

### Required Python, Ansible, and Ansible modules

The **Ansible Control Node** must have the following installed:

- **Python**
  - python >= 3.9

- **Python Modules**
  - jmespath

- **Ansible**
  - ansible-core >= 2.11, < 2.17
  - ansible: >=9.x.x

To see which Ansible version is currently installed, execute the `ansible --version` command as shown below.

#### Example: Confirming Ansible Version

  ```bash
  $ ansible --version
    ansible [core 2.12.2]
    config file = None
    configured module search path = ['/var/home/yourname/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
    ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
    ansible collection location = /var/home/yourname/.ansible/collections:/usr/share/ansible/collections
    executable location = /usr/local/bin/ansible
    python version = 3.9.9 (main, Nov 19 2021, 00:00:00) [GCC 11.2.1 20210728 (Red Hat 11.2.1-1)]
    jinja version = 3.0.3
    libyaml = True
  ```

- **Ansible Modules**: The following ansible modules are required on the control node for the
deployer to run.
  - 'ansible.posix': '>=0.0.1'
  - 'community.mongodb': '>=0.0.1'

**&#9432; Note:**
The Itential Deployer is an Ansible collection. As such, a familiarity with basic Ansible concepts
is suggested before proceeding.

### Required Public Repositories

On the Ansible control node, the Ansible Python module and the Itential Deployer Ansible collection
will need to be installed.

On the target servers, the Deployer will install RPM packages using the standard YUM repositories
and Python modules using the PyPI repository. When packages are not available for the distribution,
the Deployer will either install the required repository or download the packages.

| Component | Location | Protocol | Notes |
| :-------- | :------- | :------- | :---- |
| Ansible Control Node | <https://galaxy.ansible.com> | TCP | Itential and community Ansible collections |
| Ansible Control Node | <https://pypi.org> | TCP | Python modules |
| Itential Gateway | <https://registry.aws.itential.com> | TCP | Itential packages |
| Itential Gateway | <https://itential.jfrog.io> | TCP | Itential packages |
| Itential Gateway | <https://galaxy.ansible.com> | TCP | Community Ansible collections |
| Itential Gateway | <https://pypi.org> | TCP | Python modules |
| Itential Platform | <https://registry.aws.itential.com> | TCP | Itential packages |
| Itential Platform | <https://itential.jfrog.io> | TCP | Itential packages |
| Itential Platform | <https://registry.npmjs.org> | TCP | Core NPM packages |
| Itential Platform | <https://pypi.org> | TCP | Python modules |
| Itential Platform | <https://gitlab.com> | TCP | Platform Opensource Adapters NodeJS source repository |
| MongoDB | <https://repo.mongodb.org> | TCP | MongoDB YUM RPMs |
| MongoDB | <https://www.mongodb.org> | TCP | MongoDB YUM RPMs |
| MongoDB | <https://pgp.mongodb.org> | TCP | MongoDB YUM repository GPG Key |
| MongoDB | <https://pgp.mongodb.com> | TCP | MongoDB YUM repository GPG Key |
| MongoDB | <https://pypi.org> | TCP | Python modules |
| Redis | <http://rpms.remirepo.net> | TCP | Redis YUM RPMs<br>When installing Redis from the Remi repository |
| Redis | <https://dl.fedoraproject.org> | TCP | EPEL YUM RPMs<br>When installing Redis from the Remi repository |
| Redis | <https://github.com> | TCP | Redis source packages <br>When installing Redis from source |
| Redis | <https://codeload.github.com> | TCP | Redis source packages<br>When installing Redis from source |
| Valkey | n/a | n/a | No additional public repository required — Valkey installs via the native EL9 AppStream package, already part of the base OS's own repository set. |

If internal YUM repositories are used, refer to the
[Using Internal YUM Repositories](#using-internal-yum-repositories) section.

> [! WARNING]
> The Itential Deployer and the maintainers of the project can not know if any of the above URLs
> will result in a redirect. Reporitories may rely on CDNs or mirrors. If a customer is using a
> proxy or other such method to restrict access, this list may not represent the final URLs that are
> required.

To verify connectivity to these repositories from the target hosts (excluding the Ansible
Control Node rows, which are the control node's own responsibility), run the `verify` playbook:

```bash
ansible-playbook itential.deployer.verify -i <inventory>
```

This checks each component's target hosts (`gateway`, `platform*`, `mongodb*`, `redis_master`/
`redis_replica`, `valkey_master`/`valkey_replica`) against the rows in the table above. See
[Confirm Requirements](#confirm-requirements) for details.

### Ports and Networking

In a clustered environment where components are installed on more than one host, the following
network traffic flows need to be allowed.

| Source | Destination | Port | Protocol | Description |
| ------ | ----------- | ---- | -------- | ----------- |
| Desktop Devices | Itential Platform | 3000 | TCP | Web browser connections to Itential Platform over HTTP |
| Desktop Devices | Itential Platform | 3443 | TCP | Web browser connections to Itential Platform over HTTPS |
| Desktop Devices | IAG | 8083 | TCP | Web browser connections to IAG over HTTP |
| Desktop Devices | IAG | 8443 | TCP | Web browser connections to IAG over HTTPS  |
| Desktop Devices | Vault | 8200 | TCP | Web browser connections to Hashicorp Vault |
| Itential Platform | MongoDB | 27017 | TCP | Itential Platform connections to MongoDB |
| Itential Platform | Redis | 6379 | TCP | Itential Platform connections to Redis |
| Itential Platform | Redis | 26379 | TCP | Itential Platform connections to Redis Sentinel |
| Itential Platform | Valkey | 6379 | TCP | Itential Platform connections to Valkey |
| Itential Platform | Valkey | 26379 | TCP | Itential Platform connections to Valkey Sentinel |
| Itential Platform | IAG | 8083 | TCP | Itential Platform connections to IAG over HTTP |
| Itential Platform | IAG | 8443 | TCP | Itential Platform connections to IAG over HTTPS |
| Itential Platform | Vault | 8200 | TCP | Itential Platform connections to Hashicorp Vault |
| Itential Platform | LDAP | 389 | TCP | Itential Platform connections to LDAP when LDAP adapter is used for authentication |
| Itential Platform | LDAP | 636 | TCP | Itential Platform connections to LDAP with TLS when LDAP adapter is used for authentication |
| Itential Platform | RADIUS | 1812 | UDP | Itential Platform connections to RADIUS when RADIUS adapter is used for authentication |
| MongoDB | MongoDB | 27017 | TCP | MongoDB replication |
| Redis | Redis | 6379 | TCP | Redis replication |
| Redis | Redis | 26379 | TCP | Redis Sentinel for HA |
| Valkey | Valkey | 6379 | TCP | Valkey replication |
| Valkey | Valkey | 26379 | TCP | Valkey Sentinel for HA |

Notes

- Not all ports will need to be open for every supported architecture
- Secure ports are only required when explicitly configured in the inventory

### Certificates

The itential deployer is not responsible for creating any SSL certificates that may be used to
further tighten security in the Itential ecosystem. However, if these certificates are provided it
can upload and configure the platform to use them. The table below describes the certificates that
can be used and what their purpose is.

| Certificate | Description |
| :-----------| :-----------|
| Itential Platform webserver | Enables HTTPS communications with the Itential Platform webserver. |
| IAG webserver | Enables HTTPS communications with the IAG webserver. |
| MongoDB | Enables secure communications with the MongoDB server. Also used for intra-node mongo replication. |
| Redis | Enables secure communications with the Redis server. Also used for intra-node redis replication. |
| Valkey | Enables secure communications with the Valkey server. Also used for intra-node valkey replication. |
| LDAP | Enables secure communications with LDAP server. |

### Passwords

The deployer will create several user accounts in the dependent systems. It uses default passwords
in all cases and those passwords can be overridden with the defined ansible variables. To override
these variables just define the variable in the deployer host file.

#### MongoDB Accounts

| User Account | Default Password | Variable Name | Description |
| :----------- | :--------------- | :------------ | :---------- |
| admin | admin | mongodb_user_admin_password | Has full root access to the mongo database. |
| itential | itential | mongodb_user_itential_password | Has read and write access to the “itential” database only. |
| monitor | monitor | mongodb_user_monitor_password | Read-only access to the admin, local, and itential databases for monitoring systems. Enabled via `mongodb_monitor_user_enabled`. |

#### Redis Accounts

| User Account | Default Password | Variable Name | Description |
| :----------- | :--------------- | :------------ | :---------- |
| admin | admin | redis_user_admin_password | Has full root access to the Redis database, all channels, all keys, all commands. |
| itential | itential | redis_user_itential_password | Has full access to the Redis database, all channels, all keys, EXCEPT the following commands: asking, cluster, readonly, readwrite, bgrewriteaof, bgsave, failover, flushall, flushdb, psync, replconf, replicaof, save, shutdown, sync. |
| repluser | repluser | redis_user_repluser_password | Has access to the minimum set of commands to perform replication: psync, replconf, ping. |
| admin | sentineladmin | redis_user_sentineladmin_password | Full root access to Redis Sentinel. |
| sentineluser | sentineluser | redis_user_sentineluser_password | Has access to the minimum set of commands to perform sentinel monitoring: multi, slaveof, ping, exec, subscribe, config.rewrite, role, publish, info, client.setname, client.kill, script.kill. |
| monitor | monitor | redis_user_monitor_password | Read-only access to gather metric and cluster data from Redis and Sentinel. Enabled via `redis_monitor_user_enabled`. |

#### Valkey Accounts

| User Account | Default Password | Variable Name | Description |
| :----------- | :--------------- | :------------ | :---------- |
| admin | admin | valkey_user_admin_password | Has full root access to the Valkey database, all channels, all keys, all commands. |
| itential | itential | valkey_user_itential_password | Has full access to the Valkey database, all channels, all keys, EXCEPT the following commands: asking, cluster, readonly, readwrite, bgrewriteaof, bgsave, failover, flushall, flushdb, psync, replconf, replicaof, save, shutdown, sync. |
| repluser | repluser | valkey_user_repluser_password | Has access to the minimum set of commands to perform replication: psync, replconf, ping. |
| admin | sentineladmin | valkey_user_sentineladmin_password | Full root access to Valkey Sentinel. |
| sentineluser | sentineluser | valkey_user_sentineluser_password | Has access to the minimum set of commands to perform sentinel monitoring: multi, slaveof, ping, exec, subscribe, config.rewrite, role, publish, info, client.setname, client.kill, script.kill. |
| monitor | monitor | valkey_user_monitor_password | Read-only access to gather metric and cluster data from Valkey and Sentinel. Enabled via `valkey_monitor_user_enabled`. |

### Obtaining the Itential Binaries

#### SaaS

The latest IAG whl file is available to download from hub.itential.io.

#### On prem customers

The Itential Platform and IAG binary files are hosted on the Itential Nexus repository. An account
is required to access Itential Nexus. If you do not have an account, contact your Itential Sales
representative.

## Installing and Upgrading the Deployer

### Online Installation

The Itential Deployer can be installed via the `ansible-galaxy` utility.

On your control node, execute the following command to install the Itential Deployer:

```bash
ansible-galaxy collection install itential.deployer
```

This should also install the required ansible dependencies. When a new version of the Deployer is
available, you can upgrade using the following command:

```bash
ansible-galaxy collection install itential.deployer --upgrade
```

### Offline Installation

If your control node does not have Internet connectivity, the Itential Deployer and its
dependencies can be downloaded via another system, copied to your control node, and installed
manually.

**&#9432; Note:**
Some of the following collections may already be installed on your control node. To verify, use the
`ansible-galaxy collection list` command.

1. Download the following collections from the provided links:

    - [Itential Deployer](https://galaxy.ansible.com/ui/repo/published/itential/deployer/)
    - [Community General](https://galaxy.ansible.com/ui/repo/published/community/general/)
    - [Community MongoDB](https://galaxy.ansible.com/ui/repo/published/community/mongodb/)
    - [Ansible POSIX](https://galaxy.ansible.com/ui/repo/published/ansible/posix/)

2. Copy the downloaded collections to your control node.
3. Install the collections using the following command:

    ```bash
    ansible-galaxy collection install <COLLECTION>.tar.gz
    ```

## Running the Deployer

Once you have have installed the Itential Deployer, run it to begin deploying Itential to your
environment. This section details a basic deployment using required variables only.

### Confirm Requirements

Before running the deployer we must ensure the following:

- **Compatible OS**: Any managed nodes to be configured by the Itential Deployer must use an
operating system that is compatible with the target version of Itential Platform (and, if
applicable, IAG). For more information, refer to the [Itential Dependencies] page.
- **Hostnames**: Any hostnames used by managed nodes must be DNS-resolvable.
- **Administrative Privileges**: The `ansible` user must have administrative privileges on managed
nodes.
- **SSH Access**: The control node must have SSH connectivity to all managed nodes.

The deployer includes a playbook that can be used to confirm the environment is suitable and ready for installation. This includes checking outbound connectivity from each component's target hosts to the URLs listed in the [Required Public Repositories](#required-public-repositories) table above.

```bash
# Verify everything
ansible-playbook -i <path-to-inventory> itential.deployer.verify

# Verify Redis
ansible-playbook -i <path-to-inventory> itential.deployer.verify_redis

# Verify Valkey
ansible-playbook -i <path-to-inventory> itential.deployer.verify_valkey

# Verify MongoDB
ansible-playbook -i <path-to-inventory> itential.deployer.verify_mongodb

# Verify Platform
ansible-playbook -i <path-to-inventory> itential.deployer.verify_platform

# Verify Gateway
ansible-playbook -i <path-to-inventory> itential.deployer.verify_gateway
```

**&#9432; Note:**
Although the Itential Deployer can be used to configure nodes that use any supported operating
system, it is optimized for RHEL 8 and 9.

### Determine the Working and Deployer Directories

The Itential Deployer will be installed into the user's collection directory.  Because the Deployer
collection will be overwritten when it is upgraded, users should not store any inventory files,
binaries or artifacts in the Deployer collection directory.  Instead, users should create a working
directory to store those files.

The working directory can be any directory on the control node and will be referred to as the
`WORKING-DIR` in this guide.

Determine what directory the Itential Deployer is installed to by using the `ansible-galaxy
collection list` command. In the following example, the Deployer directory is
`/Users/<USER>/.ansible/collections/ansible_collections/itential/deployer`.

#### Example: Determining the Deployer Directory

```bash
% ansible-galaxy collection list

# /Users/<USER>/.ansible/collections/ansible_collections
Collection        Version
----------------- -------
ansible.netcommon 4.1.0
ansible.posix     1.5.4
ansible.utils     2.9.0
arista.avd        3.8.2
arista.cvp        3.6.0
arista.eos        6.0.0
community.general 7.3.0
community.mongodb 1.6.1
itential.deployer 1.0.0
```

The Deployer directory will be referred to as the `DEPLOYER-DIR` in this guide.

### Create the Inventories Directory

The `inventories` directory should be a sub-directory of the working directory. It will contain
the hosts files.

```bash
cd <WORKING-DIR>
mkdir inventories
```

### Determine Installation Artifacts Method

Choose one of the following installation methods based on your requirements:

1. **Manual Upload**: Manually download the required files onto the control node in a `files`
directory. The deployer will move these artifact files to the target nodes.
2. **Repository Download**: Provide a repository download URL with either a username/password or an
API key. The deployer will make an API request to download the files directly onto the target nodes.

### Manual Upload

#### Create the Files Directory

The `files` directory should be a sub-directory of the working directory. It will contain the
Itential binaries and artifacts.

```bash
cd <WORKING-DIR>
mkdir files
```

#### Download Installation Artifacts

Download the Itential Platform binary along with any desired Itential Platform adapters (and, if
applicable, the IAG binary) from the [Itential Nexus Repository] to local storage.

**&#9432; Note:**
If you are unsure which files should be downloaded for your environment, contact your Itential
Professional Services representative.

#### Copy Installation Artifacts into the Files Directory

Next, copy the files downloaded in the previous step to the `files` subdirectory.

#### Example: Copying to the Files Directory

```bash
cd <WORKING-DIR>/files
cp ~/Downloads/itential*.rpm .
cp ~/Downloads/automation_gateway*.whl .
```

#### Create a Symlink to the Files Directory

Navigate to the playbooks directory in the Deployer directory and create a symlink to the files
directory in the working directory.

```bash
cd <DEPLOYER-DIR>/playbooks
ln -s <WORKING-DIR>/files .
```

### Repository Download

#### Obtain the Download URL

You can obtain the download URL from either a **Sonatype Nexus Repository** or **JFrog**. Follow
the steps below based on the repository type:

- **For Sonatype Nexus**: Navigate to the file you wish to use and locate the **Path** parameter.
Copy the link provided in the **Path** field to obtain the download URL.
- **For JFrog**: Locate the file in the JFrog repository and copy the File URL.

This download method supports both the Itential Platform (bin/tar/rpm) files and the IAG (whl) files.

#### Configure Repository Credentials

Depending on the repository you are using, you will need to provide the appropriate credentials:

- **For Nexus**: Set the `repository_username` and `repository_password` variables.
- **For JFrog**: Set the `repository_api_key` variable.

**&#9432; Note:**
To secure sensitive information like passwords or API keys, consider using Ansible Vault to encrypt
these variables.

### Create the Inventory File

Using a text editor, create an inventory file that defines your deployment environment. To do this,
assign your managed nodes to the relevant groups according to what components you would like to
install on them. In the following example:

- All required variables have been defined.
- The managed node `example1.host.com` has been assigned to all groups, with the **exception** of
the `gateway` group. As such, all components **except** IAG will be installed on this node.
- The managed node `example2.host.com` has been assigned to the `gateway` group. As such, IAG will
be installed on this node.

**&#9432; Note:**
Itential recommends that all inventories follow the best practices outlined in the
[Ansible documentation](https://docs.ansible.com/ansible/latest/getting_started/get_started_inventory.html).

#### Example: Creating the Inventory File

```bash
cd <WORKING-DIR>
mkdir -p inventories/dev
vi inventories/dev/hosts
```

#### Example: Inventory File (YAML Format)

```yaml
all:
  vars:
    platform_release: 6
    # Declare the purpose of the environment to help the verification stage check resources.
    # Possible values: "dev", "test", "prod"
    env: dev

  children:
    redis_master:
      hosts:
        host01.example.com:

    mongodb:
      hosts:
        host01.example.com:

    platform:
      hosts:
        host01.example.com:
      vars:
        # open64-length hex string, representing a 256-bit AES encryption key
        # example command to generate key: 'openssl rand -hex 32'
        platform_encryption_key: <key>

        # NOTE: the platform packages list varies depending on entitlement
        platform_packages:
          - https://registry.aws.itential.com/repository/PLATFORM/Platform%206.0.0/itential-platform-<version>.noarch.rpm

        # Credentials for downloading from the Itential repository
        repository_username: <username>
        repository_password: !vault |
          $ANSIBLE_VAULT;1.1;AES123
          12341234123412341234123412341234123412341234123412341234123412341234123412341234
          12341234123412341234123412341234123412341234123412341234123412341234123412341234
          12341234123412341234123412341241234123412341234123412341234123412341234123412341
          1234123412341324

        # MongoDB connection
        platform_mongo_url: mongodb://host01.example.com:27017/itential

        # Redis connection
        platform_redis_host: host01.example.com

    gateway:
      hosts:
        host02.example.com:
      vars:
        gateway_release: 4.3
        gateway_whl_file: automation_gateway-<version>-py3-none-any.whl
```

**&#9432; Note:**
Additional example inventories can be found in the `example_inventories` directory.

### Run the Itential Deployer

Navigate to the working directory and execute the following run command.

#### Example: Running the Itential Deployer

```bash
cd <WORKING-DIR>
ansible-playbook itential.deployer.site -i inventories/dev -v
```

### Confirm Successful Installation

Deployer includes a playbook that can certify an installation and provide valuable information for
documentation purposes. Run this playbook after installation to certify an installation:

```bash
# Certify everything
ansible-playbook -i <path-to-inventory> itential.deployer.certify

# Certify Redis
ansible-playbook -i <path-to-inventory> itential.deployer.certify_redis

# Certify Valkey
ansible-playbook -i <path-to-inventory> itential.deployer.certify_valkey

# Certify MongoDB
ansible-playbook -i <path-to-inventory> itential.deployer.certify_mongodb

# Certify Platform
ansible-playbook -i <path-to-inventory> itential.deployer.certify_platform
```

This will generate markdown files one per host that contain rich detail about what was done during
the installation and also several assertions that can highlight areas of concern. These markdown
files will be copied to the control node and should be preserved as a "receipt" of installation. By
default, these files will be copied to `/tmp/itential-reports/` on the control node.

Additionally, perform the following checks on each component to confirm successful installation.

#### Itential Platform and IAG

Use a web browser to navigate to the login page of your Itential Platform/IAG servers. By default,
it is located at `http://<hostname>:3000` or `http://<hostname>:8083`, respectively. If the
Itential Platform/IAG login page is displayed, the installation was successful.

If the login page is not displayed, check that the relevant service is running on the affected
server using the `sudo systemctl status itential-platform` or
`sudo systemctl status automation-gateway` command, respectively. The output should look similar to
the following examples.

#### Example Output: Itential Platform System Status

```bash
$ sudo systemctl status itential-platform
● itential-platform.service - Itential Automation Platform Service
   Loaded: loaded (/usr/lib/systemd/system/itential-platform.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-02-01 15:21:45 UTC; 21h ago
 Main PID: 177517 (Pronghorn core)
    Tasks: 203 (limit: 23501)
   Memory: 1.0G
   CGroup: /system.slice/itential-platform.service
           ├─177517 Pronghorn core
           ├─177556 Pronghorn AGManager Application
           ├─177577 Pronghorn AdminEssentials Application
           ├─177588 Pronghorn AppArtifacts Application
           ├─177606 Pronghorn AutomationCatalog Application
           ├─177622 Pronghorn AutomationStudio Application
           ├─177659 Pronghorn ConfigurationManager Application
           ├─177674 Pronghorn FormBuilder Application
           ├─177690 Pronghorn JsonForms Application
           ├─177708 Pronghorn Jst Application
           ├─177725 Pronghorn MOP Application
           ├─177738 Pronghorn OperationsManager Application
           ├─177758 Pronghorn Search Application
           ├─177784 Pronghorn Tags Application
           ├─177800 Pronghorn TemplateBuilder Application
           ├─177820 Pronghorn WorkFlowEngine Application
           ├─177833 Pronghorn WorkflowBuilder Application
           └─177860 Pronghorn local_aaa Adapter
```

</br>

#### Example Output: IAG System Status

```bash
$ sudo systemctl status automation-gateway
● automation-gateway.service - Itential Automation Gateway
   Loaded: loaded (/etc/systemd/system/automation-gateway.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-01-17 16:48:29 UTC; 1 months 0 days ago
 Main PID: 94842 (automation-gate)
    Tasks: 10 (limit: 23435)
   Memory: 168.8M
   CGroup: /system.slice/automation-gateway.service
           ├─94842 /opt/automation-gateway/venv/bin/python3 /opt/automation-gateway/venv/bin/automation-gateway --properties-file=/etc/automation-gateway/propert>
           └─94844 /opt/automation-gateway/venv/bin/python3 /opt/automation-gateway/venv/bin/automation-gateway --properties-file=/etc/automation-gateway/propert>
```

#### MongoDB, Redis, and Valkey

From the command line of each dependency server, use the `sudo systemctl status <service>` command
to confirm that the relevant service is running. When executing the command, replace `<service>`
with one of the following:

- **MongoDB**: `mongod`
- **Redis**: `redis`
- **Valkey**: `valkey`

The output should look similar to the following examples.

#### Example Output: MongoDB Status

```bash
$ sudo systemctl status mongod
● mongod.service - MongoDB Database Server
     Loaded: loaded (/usr/lib/systemd/system/mongod.service; enabled; preset: disabled)
     Active: active (running) since Thu 2023-06-22 04:49:56 CST; 20h ago
       Docs: https://docs.mongodb.org/manual
   Main PID: 54594 (mongod)
     Memory: 156.7M
        CPU: 46.078s
     CGroup: /system.slice/mongod.service
             └─54594 /usr/bin/mongod -f /etc/mongod.conf
```

</br>

#### Example Output: Redis Status

```bash
$ sudo systemctl status redis
● redis.service - Redis persistent key-value database
     Loaded: loaded (/usr/lib/systemd/system/redis.service; enabled; preset: disabled)
    Drop-In: /etc/systemd/system/redis.service.d
             └─limit.conf
     Active: active (running) since Thu 2023-06-22 04:47:39 CST; 20h ago
   Main PID: 15723 (redis-server)
     Status: "Ready to accept connections"
      Tasks: 5 (limit: 22862)
     Memory: 9.7M
        CPU: 13.409s
     CGroup: /system.slice/redis.service
             └─15723 "/usr/bin/redis-server 127.0.0.1:6379"
```

</br>

#### Example Output: Valkey Status

```bash
$ sudo systemctl status valkey
● valkey.service - Valkey persistent key-value database
     Loaded: loaded (/usr/lib/systemd/system/valkey.service; enabled; preset: disabled)
     Active: active (running) since Thu 2026-09-11 09:15:00 UTC; 20h ago
   Main PID: 15723 (valkey-server)
     Status: "Ready to accept connections"
      Tasks: 5 (limit: 22862)
     Memory: 9.7M
        CPU: 13.409s
     CGroup: /system.slice/valkey.service
             └─15723 "/usr/bin/valkey-server 127.0.0.1:6379"
```

## Component Guides

In addition to the `itential.deployer.site` playbook, there are playbooks for each component.

Each component installed by the Itential Deployer can be granularly configured by defining
additional variables in the relevant inventory file. These additional playbooks, roles and their
corresponding variables are detailed in the following guides.

### MongoDB

[MongoDB Guide](docs/mongodb_guide.md)

### Redis

[Redis Guide](docs/redis_guide.md)

### Valkey

[Valkey Guide](docs/valkey_guide.md)

### Itential Platform

[Itential Platform Guide](docs/itential_platform_guide.md)

### Itential Gateway

[Itential Gatway Guide](docs/itential_gateway_guide.md)

## Patching Itential Platform and IAG

The Deployer supports patching Itential Platform and IAG.  Refer to the following guide for
instructions on running the patch playbooks.

[Patch Itential Platform Guide](docs/patch_itential_platform_guide.md)

[Patch IAG Guide](docs/patch_itential_gateway_guide.md)

## Using Internal YUM Repositories

By default the Deployer will install YUM repositories which point to external URLs.  If the
customer hosts repositories internally, the Deployer can be configured to skip installing the
repositories.

**&#9432; Note:**
The customer will be reposible for configuring the repo files in `/etc/yum.repos.d`.

To use internal repositories, set `common_install_yum_repos` to `false` in the `all` vars section.
For example:

```yaml
all:
  vars:
    common_install_yum_repos: false
```

## Running the Deployer in Offline Mode

The Deployer supports installations in air-gapped environments.  Refer to the following guide for
instructions on running the Deployer in offline mode.

[Offline Installation Guide](docs/offline_install_guide.md)

## Appendix A: Definition of "Highly Available" Dependencies

### Highly Available MongoDB

MongoDB clusters operate a primary/secondary model where data written to the primary will replicate
to the secondary. There is much literature on the internet about Mongo clusters. That will not be
covered here. However, it's important to note that Itential’s preferred MongoDB cluster will assume
the following requirements:

- Authentication between the replica members done with either a shared key or X.509 certificate.
- The database will have an admin user able to perform any operation.
- The database will have an “itential” user that is granted the least amount of privileges required
by the application.

Initial passwords are intended to be changed.

### Highly Available Redis

Redis clusters operate a primary/secondary model where data written to the primary will replicate
to the secondary. There is much literature on the internet about Redis clusters. That will not be
covered here. However, it's important to note that Itential’s preferred Redis cluster will assume
the following requirements:

- Authentication between the replica members is done with users defined in the Redis config file.
- Redis will have an admin user able to perform any operation.
- Redis will have an “itential” user that is granted the least amount of privileges required by the
application.
- Redis will have a replication user that is granted the least amount of privileges required by the
replication process.
- Initial passwords are intended to be changed.
- Redis Sentinel will be included to monitor the Redis cluster and will be colocated with Redis.
- Redis Sentinel will have an admin user able to perform a Sentinel task.
- Redis nodes maintain a low latency connection between nodes to avoid replication failures.

### Highly Available Valkey

Valkey is a protocol-compatible fork of Redis and its clustering model is identical: a
primary/secondary replication topology monitored by Sentinel, with the same failure-detection and
promotion mechanics described above for Redis. Itential's preferred Valkey cluster assumes the same
requirements as the preferred Redis cluster:

- Authentication between the replica members is done with users defined in the Valkey config file.
- Valkey will have an admin user able to perform any operation.
- Valkey will have an "itential" user that is granted the least amount of privileges required by the
application.
- Valkey will have a replication user that is granted the least amount of privileges required by the
replication process.
- Initial passwords are intended to be changed.
- Valkey Sentinel will be included to monitor the Valkey cluster and will be colocated with Valkey.
- Valkey Sentinel will have an admin user able to perform a Sentinel task.
- Valkey nodes maintain a low latency connection between nodes to avoid replication failures.

Unlike Redis, Valkey is currently only supported on EL9 hosts, installed exclusively via the native
AppStream package — there is no Remi option and no source-install fallback for EL8.
