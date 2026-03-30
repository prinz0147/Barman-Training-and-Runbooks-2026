# Create a standby

## Objective

Create a standby on pgnode3 using Barman. Use parallel WAL pre-fetch from `barman-wal-restore`.


## Requirements:

- rb10


## Make sure Postgres is down and PGDATA is empty on the pgnode3 server

```bash
#!/bin/bash
# Wrapping "EOF" in quotes tells the host NOT to interpret anything inside
docker exec --user postgres -i pgnode3 bash << 'EOF'
PG_DATA="/var/lib/pgsql/17/data"

# 1. Attempt to stop
/usr/pgsql-17/bin/pg_ctl -D $PG_DATA stop -m fast >/dev/null 2>&1 || true

# 2. Clear contents safely
if [ -d "$PG_DATA" ]; then
    # The '*' will now be evaluated INSIDE the container
    rm -rf $PG_DATA/*
    echo "Contents of $PG_DATA cleared."
else
    echo "Directory $PG_DATA does not exist inside the container."
fi
EOF
```


## Create a replication slot on the primary

```sql
SELECT pg_create_physical_replication_slot('pgnode3');
SELECT * FROM pg_replication_slots;
```

Output:

```
postgres=# SELECT pg_create_physical_replication_slot('pgnode3');
 pg_create_physical_replication_slot
-------------------------------------
 (pgnode3,)
(1 row)

postgres=# SELECT * FROM pg_replication_slots;
  slot_name  | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size
-------------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------
 pgnode2     |        | physical  |        |          | f         | t      |       9080 |      |              | 0/56000440  |                     | reserved   |
 barman_rb08 |        | physical  |        |          | f         | t      |       9099 |      |              | 0/56000000  |                     | reserved   |
 pgnode3     |        | physical  |        |          | f         | f      |            |      |              |             |                     |            |
(3 rows)
```


## Run the recover

The only difference to a regular recover is the addition of the `--standby-mode` argument. But let's also perform a parallel restore, using 4 workers. Note how the server is now different (pgnode3 instead of pgrestore):

```bash
barman recover \
  --remote-ssh-command "ssh postgres@pgnode3" \
  --jobs 4 \
  --standby-mode \
  rb08 latest /var/lib/postgresql/13/main/
```

The output will be similar to:

```
[barman@1c37152b8827 ~]$ barman recover \
  --remote-ssh-command "ssh postgres@pgnode3" \
  --jobs 4 \
  --standby-mode \
  rb08 latest /var/lib/pgsql/17/data/
Starting remote restore for server rb08 using backup 20260316T155148
Destination directory: /var/lib/pgsql/17/data/
Remote command: ssh postgres@pgnode3
Using safe horizon time for smart rsync copy: 2026-03-16 15:51:48.594096+00:00
Copying the base backup.
Generating recovery configuration
Identify dangerous settings in destination directory.

IMPORTANT
These settings have been modified to prevent data losses

postgresql.auto.conf line 3: archive_command = false

WARNING: 'get-wal' is in the specified 'recovery_options'.
Before you start up the PostgreSQL server, please review the postgresql.auto.conf file
inside the target directory. Make sure that 'restore_command' can be executed by the PostgreSQL user.

Restore operation completed (start time: 2026-03-17 15:45:35.061858+00:00, elapsed time: 30 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```

On server pgnode3, Barman has added data to the PGDATA directory:

```
[postgres@031a1dcd6156 data]$ ls -lh
total 100K
-rw-------. 1 postgres postgres    3 Mar 10 23:21 PG_VERSION
-rw-r--r--. 1 postgres postgres  236 Mar 16 15:51 backup_label
drwx------. 6 postgres postgres   46 Mar 10 23:26 base
-rw-------. 1 postgres postgres   30 Mar 16 00:00 current_logfiles
drwx------. 2 postgres postgres 4.0K Mar 16 15:51 global
drwx------. 2 postgres postgres    6 Mar 16 00:00 log
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_commit_ts
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_dynshmem
-rw-------. 1 postgres postgres 5.7K Mar 10 23:21 pg_hba.conf
-rw-------. 1 postgres postgres 2.6K Mar 10 23:21 pg_ident.conf
drwx------. 4 postgres postgres   68 Mar 16 15:51 pg_logical
drwx------. 4 postgres postgres   36 Mar 10 23:21 pg_multixact
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_notify
drwx------. 2 postgres postgres    6 Mar 11 00:16 pg_replslot
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_serial
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_snapshots
drwx------. 2 postgres postgres    6 Mar 11 00:15 pg_stat
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_stat_tmp
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_subtrans
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_tblspc
drwx------. 2 postgres postgres    6 Mar 10 23:21 pg_twophase
drwx------. 3 postgres postgres   28 Mar 17 15:46 pg_wal
drwx------. 2 postgres postgres   18 Mar 10 23:21 pg_xact
-rw-------. 1 postgres postgres  348 Mar 17 15:46 postgresql.auto.conf
-rw-------. 1 postgres postgres  162 Mar 11 14:48 postgresql.auto.conf.origin
-rw-------. 1 postgres postgres  31K Mar 17 15:46 postgresql.conf
-rw-------. 1 postgres postgres  31K Mar 10 23:21 postgresql.conf.origin
-rw-r--r--. 1 postgres postgres    0 Mar 17 15:46 standby.signal
```

Note how now there is a `standby.signal` file instead of a `recovery.signal` file. This means that, instead of promoting the server at the end of recovery, Postgres will stay in recovery as a standby.


## Adjust the recovery settings

These are the contents of the `postgresql.auto.conf`:

```
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
archive_mode = 'on'
#BARMAN#archive_command = 'barman-wal-archive pgbkp1 rb08 %p'
archive_command = false

# The 'barman-wal-restore' command is provided in the 'barman-cli' package
restore_command = 'barman-wal-restore -U barman 1c37152b8827 rb08 %f %p -p 4'
```

First we need to fix the `restore_command`, replacing the container ID with the container name. We will also add `-p 4`, to set parallel WAL pre-fetch to gather next 4 WAL files:

```
restore_command = 'barman-wal-restore -U barman -p 4 pgbkp1 rb08 %f %p'
```

We are also going to add the following settings so the standby is able to attach to the primary as a standby:

```
primary_conninfo = 'host=pgnode1'
primary_slot_name = 'pgnode3'
```

Setting `primary_conninfo` and `primary_slot_name` are not required if you don't need the standby to be attached to the primary. In this case, the standby will keep retrieving WAL files from Barman only through `restore_command`. In this case we say Barman is set as a WAL hub.


## Start Postgres

```bash
docker exec --user postgres pgnode3 /usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data start
```

## With an output of:

```bash
[root@localhost ~]# docker exec --user postgres pgnode3 /usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data start
waiting for server to start....2026-03-17 15:50:49.055 UTC [85136] LOG:  redirecting log output to logging collector process
2026-03-17 15:50:49.055 UTC [85136] HINT:  Future log output will appear in directory "log".
...... done
server started
```
## You might run into some issues with password like: `fe_sendauth: no password supplied`. For this, do the following:

```bash
## Create .pgpass for pgnode3:

docker exec --user postgres -i pgnode3 bash << EOF
echo "pgnode1:5432:replication:postgres:your_password_here" > ~/.pgpass
chmod 0600 ~/.pgpass
EOF
```


From log file:

```
2026-03-17 15:50:50.993 UTC [85140] LOG:  starting backup recovery with redo LSN 0/79000028, checkpoint LSN 0/79000080, on timeline ID 1
2026-03-17 15:50:53.878 UTC [85140] LOG:  restored log file "000000010000000000000079" from archive
2026-03-17 15:50:53.912 UTC [85140] LOG:  entering standby mode
2026-03-17 15:50:53.919 UTC [85140] LOG:  redo starts at 0/79000028/
2026-03-17 15:50:55.526 UTC [85140] LOG:  completed backup recovery with redo LSN 0/79000028 and end LSN 0/79000120
2026-03-17 15:50:55.526 UTC [85140] LOG:  consistent recovery state reached at 0/79000120
2026-03-17 15:50:55.526 UTC [85136] LOG:  database system is ready to accept read-only connections
ERROR: The required file is not available: 00000001000000000000007A
2026-03-17 16:02:52.469 UTC [86122] LOG:  started streaming WAL from primary at 0/7C000000 on timeline 1
2026-03-17 16:07:34.509 UTC [85138] LOG:  restartpoint starting: time
2026-03-17 16:07:34.844 UTC [85138] LOG:  restartpoint complete: wrote 4 buffers (0.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=0.329 s, sync=0.003 s, total=0.335 s; sync files=4, longest=0.002 s, average=0.001 s; distance=16387 kB, estimate=16387 kB; lsn=0/7C000CC0, redo lsn=0/7C000C68
2026-03-17 16:07:34.844 UTC [85138] LOG:  recovery restart point at 0/7C000C68
2026-03-17 16:07:34.844 UTC [85138] DETAIL:  Last completed transaction was at log time 2026-03-17 16:06:18.468157+00.
```

This means that when Postgres started, it used `restore_command` to fetch 1 WAL file from Barman.

Then trying to fetch the next WAL file failed, because it was not available on Barman (not produced by the primary or archived yet).

Then Postgres switched to `primary_conninfo` and attached to the primary.

Now on pgnode1, check the new standby is attached:

```sql
SELECT * FROM pg_stat_replication;
SELECT * FROM pg_replication_slots;
```

Output should be something like:

```
postgres=# SELECT * FROM pg_stat_replication;
SELECT * FROM pg_replication_slots;
  pid   | usesysid |     usename      |    application_name     | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn |    write_
lag    |    flush_lag    |   replay_lag    | sync_priority | sync_state |          reply_time
--------+----------+------------------+-------------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+------------+------------+------------+------------+----------
-------+-----------------+-----------------+---------------+------------+-------------------------------
 120198 |       10 | postgres         | walreceiver             | 172.18.0.6  |                 |       34990 | 2026-03-17 16:02:52.439701+00 |              | streaming | 0/7F000168 | 0/7F000168 | 0/7F000168 | 0/7F000168 |
       |                 |                 |             0 | async      | 2026-03-17 16:13:34.112774+00
 108898 |    16412 | streaming_barman | barman_receive_wal_rb08 | 172.18.0.2  |                 |       37226 | 2026-03-16 15:51:27.572285+00 |              | streaming | 0/7F000168 | 0/7F000168 | 0/7F000000 |            | 00:00:07.
656681 | 00:06:50.999045 | 24:22:05.399889 |             0 | async      | 2026-03-17 16:13:32.990539+00
 118551 |       10 | postgres         | walreceiver             | 172.18.0.5  |                 |       38964 | 2026-03-17 14:26:15.152854+00 |              | streaming | 0/7F000168 | 0/7F000168 | 0/7F000168 | 0/7F000168 |
       |                 |                 |             0 | async      | 2026-03-17 16:13:34.112461+00
(3 rows)

  slot_name  | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase | inactive_since | conflicting | invalidation
_reason | failover | synced
-------------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------+----------------+-------------+-------------
--------+----------+--------
 pgnode2     |        | physical  |        |          | f         | t      |     118551 |      |              | 0/7F000168  |                     | reserved   |               | f         |                |             |
        | f        | f
 barman_rb08 |        | physical  |        |          | f         | t      |     108898 |      |              | 0/7F000000  |                     | reserved   |               | f         |                |             |
        | f        | f
 pgnode3     |        | physical  |        |          | f         | t      |     120198 |      |              | 0/7F000168  |                     | reserved   |               | f         |                |             |
        | f        | f
(3 rows)
```
