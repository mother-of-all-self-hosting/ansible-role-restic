<!--
SPDX-FileCopyrightText: 2022 MDAD project contributors
SPDX-FileCopyrightText: 2022, 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2022-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022-2025 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up restic

This is an [Ansible](https://www.ansible.com/) role which installs [restic](https://restic.net/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Restic is a fast and secure backup program.

See the restic's [documentation](https://restic.readthedocs.io/en/stable/) to learn what restic does and why it might be useful to you.

## Prerequisites

### Set up a repository for storing backups

To use restic, it is necessary to prepare a "repository" where restic will store the backups. You can use a SFTP server, Amazon S3-compatible storage, other proprietry object storages, etc. See [this page](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html) on the official documentation for details.

When creating one, you are required to set a password (also called a key). Please take a note of it along with the repository's location, as it is necessary to provide both of them with `restic_environment_variables_restic_password` and `restic_environment_variables_restic_repository` variables, respectively.

If you are looking for an Ansible role for Rest Server, you can check out [ansible-role-restserver](https://radicle.network/nodes/iris.radicle.network/rad%3Azi4z5FpzySQ1kRqVpqcTkEfnXrD9) maintained by the [Mother-of-All-Self-Hosting (MASH)](https://github.com/mother-of-all-self-hosting) team.

## Adjusting the playbook configuration

To enable restic, add the following configuration to your `vars.yml` file (adapt to your needs).

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# restic                                                               #
#                                                                      #
########################################################################

restic_enabled: true

########################################################################
#                                                                      #
# /restic                                                              #
#                                                                      #
########################################################################
```

### Specify repository location

You also need to specify the repository's location by adding the following configuration to your `vars.yml` file:

```yaml
# Example: rest:https://example.com
restic_environment_variables_restic_repository: ""
```

### Specify password for the repository

To access the repository it is necessary to provide its password to restic by adding the following configuration to your `vars.yml` file:

```yaml
restic_environment_variables_restic_password: ""
```

### Mount a directory to backup

It is necessary to mount a data path for backing up data with the following configuration on your `vars.yml` file:

```yaml
restic_container_additional_volumes_auto:
  - type: bind
    src: PATH_TO_MOUNT_FOR_BACKUP
    dst: /data
    options: readonly
```

### Changing the pack size (optional)

Restic sets the default pack size to 16 MiB. It is possible to adjust it by adding the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
restic_environment_variables_restic_pack_size: 16
```

Refer to [this section](https://restic.readthedocs.io/en/stable/047_tuning_parameters.html#pack-size) on the official documentation for details.

### Edit the schedule (optional)

By default the task will run 4 a.m. every day based on the `restic_schedule` variable. It is defined in the format of systemd timer calendar.

To edit the schedule, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
restic_schedule: "*-*-* 04:00:00"
```

**Note**: the actual job may run with a delay. See `restic_schedule_randomized_delay_sec` for its default value.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file

See the [documentation](https://restic.readthedocs.io/en/stable/075_scripting.html#environment-variables) for a complete list of restic's config options that you could put in `restic_environment_variables_additional_variables`.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After installation, `restic backup /data` will run automatically every day at `04:00:00` (as defined in `restic_schedule` by default) to back up the directory mounted at `/data`.

### Manually start the task

Sometimes it can be helpful to run the backup as you'd like, avoiding to wait until 4 a.m., like when you test your configuration.

If you want to run it immediately, log in to the server with SSH and run `systemctl start restic` (or how you/your playbook named the service, e.g. `mash-restic`).

This will not return until the backup is done, so it can possibly take a long time. Consider using [tmux](https://en.wikipedia.org/wiki/Tmux) if your SSH connection is unstable.

### Executing commands in the container

It is possible to run a command with the command below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=command-restic -e command=COMMAND_HERE
```

For example, you can run `check` by running this command:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=command-restic -e command=check
```

## Troubleshooting

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu restic`.
