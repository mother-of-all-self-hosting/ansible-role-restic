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

########################################################################
#                                                                      #
# /restic                                                              #
#                                                                      #
########################################################################
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

### Edit the schedule (optional)

By default the task will run 4 a.m. every day based on the `restic_schedule` variable. It is defined in the format of systemd timer calendar.

To edit the schedule, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
restic_schedule: "*-*-* 04:00:00"
```

**Note**: the actual job may run with a delay. See `restic_schedule_randomized_delay_sec` on [`defaults/main.yml`](https://github.com/mother-of-all-self-hosting/ansible-role-restic/blob/f5d5b473d48c6504be10b3d946255ef5c186c2a6/defaults/main.yml#L50) for its default value.

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

After installation, `restic` will run automatically every day at `04:00:00` (as defined in `restic_schedule` by default) to back up the directory mounted at `/data`.

### Manually start the task

Sometimes it can be helpful to run the backup as you'd like, avoiding to wait until 4 a.m., like when you test your configuration.

If you want to run it immediately, log in to the server with SSH and run `systemctl start restic` (or how you/your playbook named the service, e.g. `matrix-restic`).

This will not return until the backup is done, so it can possibly take a long time. Consider using [tmux](https://en.wikipedia.org/wiki/Tmux) if your SSH connection is unstable.

## Troubleshooting

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu restic`.
