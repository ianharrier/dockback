# dockback

Backup data in Docker containers to a file share

**WARNING: This project is under construction. Some features have not yet been implemented. Please test and use at your own risk.**

### Contents

* [About](#about)
  * [Overview](#overview)
  * [Backup file naming](#backup-file-naming)
* [Installing](#installing)
* [Configuration options](#configuration-options)
  * [Job types](#job-types)
    * [`compress_directory`](#compress_directory)
    * [`file_copy`](#file_copy)
    * [`mysql_dump`](#mysql_dump)
    * [`postgres_dump`](#postgres_dump)
* [Un-installing](#un-installing)
* [Future work](#future-work)

## About

### Overview

dockback backs up data from various Docker containers to a Windows (CIFS) file share. It was written for Ubuntu 16.04 LTS but will probably work on other systems with few modifications. Where applicable, backups are compressed using `zip` for better compatibility with Windows systems.

It is assumed that all containers are using volumes to store their data somewhere on the host (i.e. directly accessible by dockback). With the exception of database dumps, dockback will not "reach inside" containers to backup data.

### Backup file naming

When naming backup files, dockback appends an ISO 8601 compliant timestamp to the job name. For example:

* `webapp-db_20161219T201720-0600.sql`
* `webapp_20161219T201740-0600.zip`

## Installing

To install a specific version of dockback, run the following commands.

```bash
sudo git clone https://github.com/ianharrier/dockback.git /opt/dockback
sudo chmod +x /opt/dockback/dockback
sudo ln -s /opt/dockback/dockback /usr/local/sbin/dockback

# List available versions
git -C /opt/dockback tag

# Checkout the desired version (e.g. 0.2.0)
sudo git -C /opt/dockback checkout tags/v0.2.0
```

After dockback has been installed, you should modify the configuration file at `/opt/dockback/config`. Configuration options can be found below.

You should also check if the dependencies are installed with the following command.

```bash
dockback test dep
```

Note that dockback will use the system mail configuration. On Ubuntu 16.04 (and others), for example, you can configure postfix to forward all emails to an external unauthenticated SMTP relay by running the following commands, replacing `mail.domain.tld` with the FQDN of your SMTP relay.

```bash
# Preselect the configuration options for postfix
sudo debconf-set-selections <<EOF
postfix postfix/mailname string $(hostname --fqdn)
postfix postfix/main_mailer_type select Satellite system
postfix postfix/relayhost string mail.domain.tld
EOF
# Install mailutils (which includes postfix)
sudo apt -y install mailutils
```

## Configuration options

* `share_remote_path` UNC path of the remote CIFS share (e.g. `'//server/share/path'`). May contain unescaped spaces (e.g. `'//server/share/folder with spaces'`).
* `share_username` User who has read/write access to `share_remote_path`.
* `share_password` Alphanumeric password for `share_username`.
* `share_domain` Domain name for `share_username`. This variable is optional; leave blank if not in a domain environment.
* `email_enabled` Whether email notifications are sent prior to exiting. Valid options are `'true'` and `'false'`.
* `email_from` Email address from which notification emails will be sent. Should be formatted as `'username@domain.tld'`.
* `email_to` Email address to which notification emails will be sent. Should be formatted as `'username@domain.tld'`.
* `email_subject` Subject on the notification email.
* `backup_frequency_hours` The frequency in hours at which `cron` will run `dockback up all`.
* `backup_jobs` Array containing the backup job strings, one per element. Job strings should be formatted as `'key1=value1|key2=value2|key3=value3'` and should contain the following required keys:
  * `job_name` Name of the backup job. Usually similar or identical to the container name. For example: `'job_name=container1|...'`
  * `job_type` Type of backup job. For example: `'job_name=container1|job_type=compress_directory|...'`

### Job types

dockback supports a variety of backup types. One type is required per job.

#### `compress_directory`

Compress the files(s) in a directory on the Docker host [which is mounted to a container as a volume]. This job type is designed for containers that do not need special considerations to ensure the backup is application consistent. For example, this job type can be used to backup a directory storing uploads from a web form, but it should not be used for most databases.

Job strings should contain the following required key:

* `directory_path` Path of the directory to be compressed.

Example job string using all options:

`'job_name=container1|job_type=compress_directory|directory_path=/docker-volumes/container1'`

#### `file_copy`

Copy the file(s) in a directory on the Docker host [which is mounted to a container as a volume]. This job type is designed for containers which produce their own backup files and store them in a directory.

Job strings should contain the following required key:

* `file_path` Path to the directory containing files to be copied. Must contain a wildcard (`*`) for the file name and/or extension.

Job strings may contain the following optional keys:

* `delete_after_copy` Optionally delete the original file(s). In other words, move (rather than copy) the file(s) to the backup destination. This option is designed for containers that do not automatically delete old backup files. Valid options are `true` and `false`, and the default option is `false`.
* `max_age_minutes` Only copy file(s) younger than the specified number of minutes. This option is useful when (a) `delete_after_copy` is `false` and (b) the container automatically deletes old backup files, as to not copy previously-copied files.

Example job string using all options:

`'job_name=container1|job_type=file_copy|directory_path=/docker-volumes/container1/subfolder/*.zip|delete_after_copy=true|max_age_minutes=1440'`

#### `mysql_dump`

Dump one or all databases in an [official MySQL container](https://hub.docker.com/_/mysql/). This job type assumes the `MYSQL_ROOT_PASSWORD` environment variable has been set on the container.

Job strings should contain the following required key:

* `container_name` Name of the MySQL container.

Job strings may contain the following optional keys:

* `db` Name of the database to be backed up. If this option is not set, all databases will be backed up.
* `compress` Compress the database dump. If this option is `false` or is not set, the dump will be saved as a `sql` file. If this option is `true`, the `sql` file will be compressed.

Example job string using all options:

`'job_name=container1|job_type=mysql_dump|container_name=container1|db=exampledb|compress=true'`

#### `postgres_dump`

Dump one or all databases in an [official PostgreSQL container](https://hub.docker.com/_/postgres/).

Job strings should contain the following required key:

* `container_name` Name of the PostgreSQL container.

Job strings may contain the following optional keys:

* `db` Name of the database to be backed up. If this option is not set, all databases will be backed up.
* `compress` Compress the database dump. If this option is `false` or is not set, the dump will be saved as a `sql` file. If this option is `true`, the `sql` file will be compressed.

Example job string using all options:

`'job_name=container1|job_type=postgres_dump|container_name=container1|db=exampledb|compress=true'`

## Un-installing

Before uninstalling dockback, run the following command to verify the backup share has been unmounted.

```bash
sudo umount /mnt/shares/dockback
```

To uninstall dockback, run the following commands.

```bash
sudo rm -rf /opt/dockback
sudo rm /usr/local/sbin/dockback
sudo rm /etc/cron.d/dockback
sudo rmdir /mnt/shares/dockback
sudo rm -rf /tmp/dockback.*
```

## Future work

The following features might be added in a future release, in no particular order.

1. Only check for requirements needed per user configuration.
2. Allow file share protocols other than CIFS.
3. Allow compression formats other than ZIP.
4. Allow custom scheduling options.
5. Check for mail errors.
6. Ability to backup data by "reaching inside" containers (i.e. without using volumes).
7. Add a compress option to `file_copy`, in case backup files are not already compressed.
8. Allow dockback to run from a Docker container.
9. Check the dockback configuration file for errors.
10. Handle the configuration file differently. For example:
  * Store in another directory
  * Utilize `.gitignore` file
  * Include a `config.example` file
