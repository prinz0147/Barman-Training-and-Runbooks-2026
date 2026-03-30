# Point Barman to a Standby

## Objective

Setup Barman to take backups from the Postgres standby, with rsync backup and 2 WAL methods.


## Requirements:

- rb04


## Create configuration file

Create a file called `/etc/barman.d/rb09.conf` with the following contents:

```ini
[rb09]

description = "PostgreSQL standby database - Runbook 09"
ssh_command = ssh -q postgres@pgnode2
conninfo = host=pgnode2 user=barman dbname=postgres
streaming_conninfo = host=pgnode2 user=streaming_barman

backup_method = rsync
reuse_backup = link
backup_options = concurrent_backup
parallel_jobs = 4
network_compression = false
immediate_checkpoint = true

last_backup_maximum_age = 2 DAYS
minimum_redundancy = 2

retention_policy = RECOVERY WINDOW OF 1 WEEK

archiver = on
archiver_batch_size = 50

streaming_archiver = on
slot_name = barman_rb09
create_slot = auto
streaming_archiver_name = barman_receive_wal_rb09
streaming_archiver_batch_size = 50

recovery_options = get-wal
path_prefix = "/usr/pgsql-17/bin"
active = true
```


## Check configuration

Become `barman` user:

```bash
su - barman
```

Check server status:

```bash
barman check rb09
```

Output will be:

```
barman@2534c27e5636:~$ barman check rb09
Server rb09:
        WAL archive: FAILED (please make sure WAL shipping is setup)
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: FAILED (replication slot 'barman_rb09' doesn't exist. Please execute 'barman receive-wal --create-slot rb09')
        directories: OK
        retention policy settings: OK
        backup maximum age: FAILED (interval provided: 2 days, latest backup age: No available backups)
        backup minimum size: OK (0 B)
        wal maximum age: OK (no last_wal_maximum_age provided)
        wal size: OK (0 B)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: FAILED (have 0 non-incremental backups, expected at least 2)
        ssh: OK (PostgreSQL server)
        systemid coherence: OK (no system Id stored on disk)
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: FAILED (See the Barman log file for more details)
        archive_mode: FAILED (please set it to 'on' or 'always')
        archive_command: FAILED (please set it accordingly to documentation)
        archiver errors: OK
```


## Set archive_mode and archive_command on Postgres

```bash
# Update pgnode2 to archive even while in recovery
cat << EOF | docker exec --user=postgres -i pgnode2 /usr/pgsql-17/bin/psql -h /tmp
ALTER SYSTEM SET archive_mode = 'always';
ALTER SYSTEM SET archive_command = 'barman-wal-archive pgbkp1 rb09 %p';
EOF

# Restart to apply the 'always' mode
docker exec --user=postgres -i pgnode2 /usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data restart
```

Now run `barman check rb09` again.


## Create the replication slot for Barman

```bash
barman receive-wal --create-slot rb09
```

Output would be:

```
barman@9c99447bf70f:~$ barman receive-wal --create-slot rb09
Creating physical replication slot 'barman_rb09' on server 'rb09'
Replication slot 'barman_rb09' created
```

Now run `barman check rb09` again.

Also check `pg_replication_slots` on the standby, and confirm the slot exists, but it's still inactive.



## Start pg_receivewal on Barman

```bash
barman cron
```

Output should be:

```
barman@9c99447bf70f:~$ barman cron
Starting WAL archiving for server rb09
Starting streaming archiver for server rb09
```

Check `pg_receivewal` is running. Now run `barman check rb09` again.

If the server has no load, you will still see:

```
WAL archive: FAILED (please make sure WAL shipping is setup)
```


## Run switch-wal

Try to run the following:

```bash
barman switch-wal --archive --archive-timeout 60 rb09
```

Output will be:

```
barman@9c99447bf70f:~$ barman switch-wal --archive --archive-timeout 60 rb09
No switch performed because server 'rb09' is a standby.
Waiting for a WAL file from server 'rb09' to be archived (max: 60 seconds)
A WAL file has not been received in 60 seconds
```

Which means it can't run `pg_switch_wal()` on the standby. So you need to run that against the primary, twice:

```
psql -h pgnode1 -d postgres -c 'SELECT pg_switch_wal()'
psql -h pgnode1 -d postgres -c 'SELECT pg_switch_wal()'
```

Run `barman cron`, then run `barman check rb09` again.


## You may need to install `rsync`  (OPTIONAL)

```bash

#!/bin/bash

# Define all containers
CONTAINERS=("pgnode1" "pgnode2" "pgnode3" "pgrestore" "pgbkp1" "pgbkp2")

echo "--- Installing rsync on all nodes ---"

for container in "${CONTAINERS[@]}"
do
    echo -n "$container: "
    # Install rsync silently (-q) and confirm success
    docker exec -u root $container dnf install -y -q rsync && echo "INSTALLED" || echo "FAILED"
done

echo "--- Verification ---"
for container in "${CONTAINERS[@]}"
do
    docker exec $container rsync --version | head -n 1
done
```

## Run 2 base backups

```bash
barman backup --jobs 4 rb09
barman backup --jobs 4 rb09
```


Each backup might have stuck on `Asking PostgreSQL server to finalize the backup.`. In this case, go to the primary and run `SELECT pg_switch_wal()` twice.

Then run `barman check rb09` again. It should be all green.

Also run `barman list-backup rb09`:

```
barman@9c99447bf70f:~$ barman list-backup rb09
rb09 20210816T191222 - Mon Aug 16 19:12:23 2021 - Size: 1.6 GiB - WAL Size: 0 B
rb09 20210816T190710 - Mon Aug 16 19:07:18 2021 - Size: 1.5 GiB - WAL Size: 48.0 MiB
```



## **Please note that you may encounter some minor SSH issues. If you are unable to resolve it and your backup hangs, please PING me on the training SLACK Channel**