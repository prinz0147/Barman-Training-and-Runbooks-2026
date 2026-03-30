# Geo-Redundancy

## Objective

Configure `pgbkp2` as a Passive Barman server of `pgbkp1` and verify the Geo-Redundancy features.


## Requirements:

- rb10
- PG 17 Cluster (`/var/lib/pgsql/17/data`)


## Setup Passive Barman in pgbkp2

As root, create a file called `/etc/barman.d/rb15.conf` with the following content:

```
[rb08]
description = "Passive Barman server for the PostgreSQL primary"
primary_ssh_command = ssh -q barman@pgbkp1
reuse_backup = link
```

Note that despite of the configuration file name (`rb15.conf`), we need to set the server name as `rb08`, which is the name of the configuration server on the Primary Barman server.

After creating the configuration file, check `barman list-server`, you will see this:

```
[barman@1c37152b8827 ~]$ barman list-server
rb08 - Passive Barman server for the PostgreSQL primary (Passive)
```

Meaning this is a Passive server. You can have mixed Passive and Direct servers in the same Barman server.

Now let's run `barman cron` to start retrieving information from the Primary Barman:

```
[barman@1c37152b8827 ~]$ barman cron
Starting copy of backup 20260311T150418 for server rb08
Started copy of WAL files for server rb08
```

After a short while, you will see that the first backup is copied:

```
[barman@1b82f93e7ea5 barman.d]$ barman list-backup rb08
rb08 20260311T150418 - F - SYNCING

[barman@1b82f93e7ea5 barman.d]$ barman list-backup rb08
rb08 20260311T150418 - F - Wed Mar 11 15:04:20 2026 - Size: 1.5 GiB - WAL Size: 0 B
```

You can wait for further executions of the scheduled `barman cron`, or execute `barman cron` manually:

```
barman@a275b5e16ebc:~$ barman cron
Starting copy of backup 20210818T143622 for server rb08
Started copy of WAL files for server rb08
```

Eventually the backup and WAL catalog on the Passive server will be in sync with the Primary server:

```
[barman@1b82f93e7ea5 barman.d]$ barman list-backup rb08
rb08 20260311T154518 - F - Wed Mar 11 15:45:20 2026 - Size: 1.5 GiB - WAL Size: 352.0 MiB
rb08 20260311T150418 - F - Wed Mar 11 15:04:20 2026 - Size: 1.5 GiB - WAL Size: 128.0 MiB
```


## How this works

Every time `barman cron` is executed, it runs the following command:

```bash
barman sync-info --primary rb08
```

To get the inventory of backups and WAL files from the Primary Barman. Then it runs:

```bash
barman sync-info rb08
```

To gather the inventory of backups and WAL files from the Passive Barman.

Comparing the inventories, Barman deletes backups and WAL files that were deleted from the Primary Server, and copies backups and WAL files that were included in the Primary Server.

The following command is used to copy backups:

```bash
barman sync-backup <server> <backup_id>
```

And the following command is used to copy WAL files:

```bash
barman sync-wals <server>
```

These commands can also be executed manually.


## Create a new backup on the Primary Barman

On pgbkp1, currently we have 2 backups only:

```
barman@5ddecfa6e4d2:~/rb08$ barman list-backup rb08
rb08 20210818T143622 - Wed Aug 18 14:36:23 2021 - Size: 1.5 GiB - WAL Size: 32.0 MiB
rb08 20210818T142757 - Wed Aug 18 14:28:06 2021 - Size: 1.5 GiB - WAL Size: 48.0 MiB
```

Let's take another backup:

```
barman backup rb08
```

Once the backup is finished on pgbpk1, then on pgbkp2 we can either wait for another scheduled execution of `barman cron` or run `barman cron` ourselves.

Then check `barman list-backup rb08` in the Passive Barman server again.

