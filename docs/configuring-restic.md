<!--
SPDX-FileCopyrightText: 2022 MDAD project contributors
SPDX-FileCopyrightText: 2022, 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2022-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022-2025 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up restic

This is an [Ansible](https://www.ansible.com/) role which installs and configures [restic](https://www.restic.org/) (short: restic) with [restic](https://torsion.org/restic/) in a [Docker](https://www.docker.com/) container wrapped in a systemd service.

restic is a deduplicating backup program with optional compression and encryption. That means your daily incremental backups can be stored in a fraction of the space and is safe whether you store it at home or on a cloud service.

See the restic's [documentation](https://torsion.org/restic/reference/configuration/) to learn what restic does and why it might be useful to you.

## Prerequisites

### Set up a remote server for storing backups

You will need a remote server where restic will store the backups. There are hosted, restic compatible solutions available, such as [resticBase](https://www.resticbase.com).

### Check the Postgres version

For some playbooks, if you're using the integrated Postgres database server, backups with restic will also include dumps of your Postgres database by default.

Unless you disable the Postgres-backup support, make sure that the Postgres version of your homeserver's database is compatible with restic. You can check the compatible versions on [`defaults/main.yml`](../defaults/main.yml).

An alternative solution for backing up the Postgres database is [Postgres backup](https://github.com/mother-of-all-self-hosting/ansible-role-postgres-backup). If you decide to go with another solution, you can disable Postgres-backup support for restic using the `restic_postgresql_enabled` variable.

### Create a new SSH key

Run the command below on any machine to create a new SSH key:

```bash
ssh-keygen -t ed25519 -N '' -f restic-backup -C restic-backup
```

You don't need to place the key in the `.ssh` folder.

### Add the public key

Next, add the **public** part of this SSH key (the `restic-backup.pub` file) to your restic provider/server.

If you are using a hosted solution, follow their instructions. If you have your own server, copy the key to it with the command like below:

```sh
# Example to append the new PUBKEY contents, where:
# - PUBKEY is path to the public key
# - USER is a SSH user on a provider / server
# - HOST is a SSH host of a provider / server
cat PUBKEY | ssh USER@HOST 'dd of=.ssh/authorized_keys oflag=append conv=notrunc'
```

The **private** key needs to be added to `restic_ssh_key_private` on your `vars.yml` file as below.

## Adjusting the playbook configuration

To enable restic, add the following configuration to your `vars.yml` file (adapt to your needs).

**Note**: the path should be something like `inventory/host_vars/matrix.example.com/vars.yml` if you use the [matrix-docker-ansible-deploy (MDAD)](https://github.com/spantaleev/matrix-docker-ansible-deploy) or [Mother-of-All-Self-Hosting (MASH)](https://github.com/mother-of-all-self-hosting/mash-playbook) Ansible playbook.

```yaml
########################################################################
#                                                                      #
# restic                                                               #
#                                                                      #
########################################################################

restic_enabled: true

# Set the repository location, where:
# - USER is a SSH user on a provider / server
# - HOST is a SSH host of a provider / server
# - REPO is a restic repository name
restic_location_repositories:
 - ssh://USER@HOST/./REPO

# Generate a strong password used for encrypting backups. You can create one with a command like `pwgen -s 64 1`.
restic_storage_encryption_passphrase: "PASSPHRASE"

# Add the content of the **private** part of the SSH key you have created.
# Note: the whole key (all of its belonging lines) under the variable needs to be indented with 2 spaces.
restic_ssh_key_private: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  TG9yZW0gaXBzdW0gZG9sb3Igc2l0IGFtZXQsIGNvbnNlY3RldHVyIGFkaXBpc2NpbmcgZW
  xpdCwgc2VkIGRvIGVpdXNtb2QgdGVtcG9yIGluY2lkaWR1bnQgdXQgbGFib3JlIGV0IGRv
  bG9yZSBtYWduYSBhbGdlxdWEuIFV0IGVuaW0gYWQgbWluaW0gdmVuaWFtLCBxdWlzIG5vc3
  RydWQgZXhlcmNpdGF0aW9uIHVsbGFtY28gbGFib3JpcyBuaXNpIHV0IGFsaXF1aXAgZXgg
  ZWEgY29tbW9kbyBjb25zZXF1YXQuIA==
  -----END OPENSSH PRIVATE KEY-----

########################################################################
#                                                                      #
# /restic                                                              #
#                                                                      #
########################################################################
```

**Note**: `REPO` will be initialized on backup start, for example: `matrix`. See [Remote repositories](https://restic.readthedocs.io/en/stable/usage/general.html#repository-urls) for the syntax.

### Set backup archive name (optional)

You can specify the backup archive name format. To set it, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
restic_storage_archive_name_format: restic-{now:%Y-%m-%d-%H%M%S}
```

### Configure retention policy (optional)

It is also possible to configure a retention strategy. To configure it, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
restic_retention_keep_hourly: 0
restic_retention_keep_daily: 7
restic_retention_keep_weekly: 4
restic_retention_keep_monthly: 12
restic_retention_keep_yearly: 2
```

### Edit the schedule (optional)

By default the task will run 4 a.m. every day based on the `restic_schedule` variable. It is defined in the format of systemd timer calendar.

To edit the schedule, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
restic_schedule: "*-*-* 04:00:00"
```

**Note**: the actual job may run with a delay. See `restic_schedule_randomized_delay_sec` on [`defaults/main.yml`](https://github.com/mother-of-all-self-hosting/ansible-role-restic/blob/f5d5b473d48c6504be10b3d946255ef5c186c2a6/defaults/main.yml#L50) for its default value.

### Set include and/or exclude directories (optional)

`restic_location_source_directories` defines the list of directories to back up.

You might also want to exclude certain directories or file patterns from the backup using the `restic_location_exclude_patterns` variable.

>[!NOTE]
> If you use multiple playbooks which utliize this role, you should make sure by yourself that the all of the directories to be backed up which those playbooks manage are correctly specified to `restic_location_source_directories`.
>
> For example, backing up data directories of the [matrix-docker-ansible-deploy Ansible playbook](https://github.com/spantaleev/matrix-docker-ansible-deploy) and the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook) requires both `matrix_base_data_path` and `mash_playbook_base_path` to be specified. Refer to the configuration files of the playbooks for details.

### Configuring ntfy integration (optional)

You can also have the service send push notifications to your self-hosted [ntfy](https://ntfy.sh/) instance. To enable it, it is necessary to specify the topic to send them, the server's hostname, and login credentials (username and password, or access token) by adding the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
restic_ntfy_topic: YOUR_NTFY_TOPIC_HERE
restic_ntfy_server: YOUR_NTFY_INSTANCE_HOSTNAME_HERE

# Specify username and password
restic_ntfy_access_username: ""
restic_ntfy_access_password: ""

# Otherwise, specify the access token
restic_ntfy_access_token: ""
```

Refer to [this page](https://torsion.org/restic/reference/configuration/monitoring/ntfy/) on the official documentation for details.

If you are looking for an Ansible role for ntfy, you can check out [ansible-role-ntfy](https://github.com/mother-of-all-self-hosting/ansible-role-ntfy) maintained by the [Mother-of-All-Self-Hosting (MASH)](https://github.com/mother-of-all-self-hosting) team.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `restic_configuration_extension_yaml` variable

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MDAD / MASH playbook, the shortcut commands with the [`just` program](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After installation, `restic` will run automatically every day at `04:00:00` (as defined in `restic_schedule` by default).

### Manually start the task

Sometimes it can be helpful to run the backup as you'd like, avoiding to wait until 4 a.m., like when you test your configuration.

If you want to run it immediately, log in to the server with SSH and run `systemctl start restic` (or how you/your playbook named the service, e.g. `matrix-restic`).

This will not return until the backup is done, so it can possibly take a long time. Consider using [tmux](https://en.wikipedia.org/wiki/Tmux) if your SSH connection is unstable.

### Example playbook

```yaml
- hosts: servers
  roles:
    - role: galaxy/com.devture.ansible.role.systemd_docker_base

    # This role is not required. We just use it in our example.
    - role: galaxy/postgres

    - role: galaxy/ansible.role.restic

    - role: another_role
```

Example playbook configuration (`group_vars/servers` or other):

>[!NOTE]
> The configuration below wires the restic role with [this MASH/Postgres role](https://github.com/mother-of-all-self-hosting/ansible-role-postgres). Note that this is just an example. You can use this role without it Postgres integration or with another Postgres instance.

```yaml
restic_enabled: false

restic_identifier: my-restic

restic_base_path: "{{ my_base_path }}/restic"

restic_username: "{{ my_username }}"
restic_uid: "{{ my_uid }}"
restic_gid: "{{ my_gid }}"

# We assume Postgres is installed via the `com.devture.ansible.role.postgres` role.
# Remove this and any `postgres_*` reference below, if that's not the case.
restic_postgresql_version_detection_postgres_role_name: galaxy/com.devture.ansible.role.postgres

# If you will use this without `com.devture.ansible.role.postgres`, you'll need to set the major Postgres version manually instead.
# restic_postgres_version: 15

restic_container_network: "{{ postgres_container_network if postgres_enabled else restic_identifier }}"

restic_container_image_self_build: "{{ architecture not in ['amd64', 'arm32', 'arm64'] }}"

restic_postgresql_enabled: "{{ postgres_enabled }}"
restic_postgresql_databases_hostname: "{{ postgres_connection_hostname if postgres_enabled else '' }}"
restic_postgresql_databases_username: "{{ postgres_connection_username if postgres_enabled else '' }}"
restic_postgresql_databases_password: "{{ postgres_connection_password if postgres_enabled else '' }}"
restic_postgresql_databases_port: "{{ postgres_connection_port if postgres_enabled else 5432 }}"
restic_postgresql_databases: "{{ postgres_managed_databases | map(attribute='name') if postgres_enabled else [] }}"

restic_location_source_directories:
  - "{{ my_data_path }}"

restic_systemd_required_services_list_auto: |
  {{
    ([postgres_identifier ~ '.service'] if postgres_enabled else [])
  }}
```

## Troubleshooting

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu restic`.
