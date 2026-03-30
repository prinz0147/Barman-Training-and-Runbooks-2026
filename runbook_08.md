# rsync backup with 2 WAL methods

## Objective

Setup Barman to take backups from the Postgres primary, with rsync backup and 2 WAL methods.


## Requirements:

- rb04


## Create configuration file

As root, create a file called `/etc/barman.d/rb08.conf` with the following contents:

```ini
[rb08]
description = "PostgreSQL 17 primary - Runbook 08"
ssh_command = ssh -q postgres@pgnode1
conninfo = host=pgnode1 user=barman dbname=postgres
streaming_conninfo = host=pgnode1 user=streaming_barman

# Backup Strategy
backup_method = rsync
reuse_backup = link
backup_options = concurrent_backup
parallel_jobs = 4
network_compression = false
immediate_checkpoint = true

# Retention Policies
last_backup_maximum_age = 2 DAYS
minimum_redundancy = 2
retention_policy = RECOVERY WINDOW OF 1 WEEK

# WAL Method 1: Archiver (via archive_command)
archiver = on
archiver_batch_size = 50

# WAL Method 2: Streaming Archiver (via replication slot)
streaming_archiver = on
slot_name = barman_rb08
create_slot = auto
streaming_archiver_name = barman_receive_wal_rb08
streaming_archiver_batch_size = 50

# Recovery and Paths
recovery_options = get-wal
path_prefix = "/usr/pgsql-17/bin/"
active = true
```


## Check configuration

Become `barman` user:

```bash
su - barman
```

List servers with:

```bash
barman list-server
```

Output should be:

```
[barman@7314d2550ec8 ~]$ barman list-server
rb08 - PostgreSQL 17 primary - Runbook 08
```

Check server status:

```bash
barman check rb08
```

Output should be:

```
root@1dc73c3167f9:~# barman check rb08
Server rb08:
	WAL archive: FAILED (please make sure WAL shipping is setup)
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: FAILED (replication slot 'barman_rb08' doesn't exist. Please execute 'barman receive-wal --create-slot rb08')
	directories: OK
	retention policy settings: OK
	backup maximum age: FAILED (interval provided: 2 days, latest backup age: No available backups)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: FAILED (have 0 backups, expected at least 2)
	ssh: OK (PostgreSQL server)
	not in recovery: OK
	systemid coherence: OK (no system Id stored on disk)
	pg_receivexlog: OK
	pg_receivexlog compatible: OK ----> (NOTE: You might run into some issue but you will need to install PG on the barman node. Thats it)
	receive-wal running: FAILED (See the Barman log file for more details)
	archive_mode: FAILED (please set it to 'on' or 'always')
	archive_command: FAILED (please set it accordingly to documentation)
	archiver errors: OK
```

As you can see there are several configuration issues! We should now fix them one by one.


Other commands to try:

```
barman status rb08
barman show-server rb08
```


## Set archive_mode and archive_command on Postgres

```bash
# 1. Set the configuration as before
cat << EOF | docker exec --user=postgres -i pgnode1 psql
ALTER SYSTEM SET archive_mode = on;
ALTER SYSTEM SET archive_command = 'barman-wal-archive --ssh pgbkp1 rb08 %p';
EOF

# 2. Restart using pg_ctl (RHEL Standard)
# Replace '/var/lib/pgsql/17/data' with your actual PGDATA path if different
docker exec --user=postgres -i pgnode1 /usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data restart
```

Now run `barman check rb08` again.


## Create the replication slot for Barman

```bash
barman receive-wal --create-slot rb08
```

Output would be:

```
barman@9c99447bf70f:~$ barman receive-wal --create-slot rb08
Creating physical replication slot 'barman_rb08' on server 'rb08'
Replication slot 'barman_rb08' created
```

Now run `barman check rb08` again.

Also check `pg_replication_slots` on Postgres, and confirm the slot exists, but it's still inactive.

Also try the following command:

```bash
barman replication-status rb08
```


## Start pg_receivewal on Barman

```bash
barman cron
```

Output should be:

```
barman@9c99447bf70f:~$ barman cron
Starting WAL archiving for server rb08
Starting streaming archiver for server rb08
```

Check `pg_receivewal` is running:

```
[barman@7314d2550ec8 ~]$ ps aux | grep receive
barman      2773  0.7  1.0  45748 38584 ?        Ss   20:19   0:00 /usr/bin/python3.12 /usr/bin/barman -c /etc/barman.conf -q receive-wal rb08
barman      2775  0.0  0.2  16172  9280 ?        S    20:19   0:00 /usr/pgsql-17/bin/pg_receivewal --dbname=dbname=replication host=pgnode1 options=-cdatestyle=iso replication=true user=streaming_barman application_name=barman_receive_wal_rb08 --verbose --no-loop --no-password --directory=/var/lib/barman/rb08/streaming --slot=barman_rb08
barman      2812  0.0  0.0   3728  2120 pts/2    S+   20:21   0:00 grep --color=auto receive
```

Now run `barman check rb08` again.

If the server has no load, you will still see:

```
WAL archive: FAILED (please make sure WAL shipping is setup)
```


## Run switch-wal

```bash
barman switch-wal --archive --archive-timeout 60 rb08
```

**Note:** Sometimes, `rsync` may not come out of the box which may lead to the backup hanging. Ensure it is installed and that should solve the issue.

Output will be:

```
[barman@7314d2550ec8 ~]$ barman switch-wal --archive --archive-timeout 60 rb08
The WAL file 000000010000000000000057 has been closed on server 'rb08'
Waiting for the WAL file 000000010000000000000057 from server 'rb08' (max: 60 seconds)
Processing xlog segments from streaming for rb08 (batch size: 1)
        000000010000000000000057
```

Run `barman replication-status rb08` again, output will be:

```
[barman@7314d2550ec8 ~]$ barman replication-status rb08
Status of streaming clients for server 'rb08':
  Current LSN on master: 0/58000060
  Number of streaming clients: 2

  1. Async standby
     Application name: walreceiver
     Sync stage      : 5/5 Hot standby (max)
     Communication   : TCP/IP
     IP Address      : 172.18.0.5 / Port: 50196 / Host: -
     User name       : postgres
     Current state   : streaming (async)
     Replication slot: pgnode2
     WAL sender PID  : 3079
     Started at      : 2026-03-05 20:19:29.867402+00:00
     Sent LSN   : 0/58000060 (diff: 0 B)
     Write LSN  : 0/58000060 (diff: 0 B)
     Flush LSN  : 0/58000060 (diff: 0 B)
     Replay LSN : 0/58000060 (diff: 0 B)

  2. Async WAL streamer
     Application name: barman_receive_wal_rb08
     Sync stage      : 3/3 Remote write
     Communication   : TCP/IP
     IP Address      : 172.18.0.2 / Port: 52030 / Host: -
     User name       : streaming_barman
     Current state   : streaming (async)
     Replication slot: barman_rb08
     WAL sender PID  : 3093
     Started at      : 2026-03-05 20:19:41.390407+00:00
     Sent LSN   : 0/58000060 (diff: 0 B)
     Write LSN  : 0/58000060 (diff: 0 B)
     Flush LSN  : 0/58000000 (diff: -96 B)
```

Now run `barman check rb08` again. You will finally see this:

```
[barman@7314d2550ec8 ~]$ barman check rb08
Server rb08:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
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
        receive-wal running: OK
        archive_mode: OK
        archive_command: OK
        archiver errors: OK
```

As you can see, the only `FAILED` items are about backup (backup maximum age and minimum redundancy requirements). In the next runbook we will see how to take a backup.
