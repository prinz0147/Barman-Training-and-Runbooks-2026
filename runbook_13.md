# PITR

## Objective

Perform a PITR using Barman.


## Requirements:

- rb10


## Make sure Postgres is down and PGDATA is empty on the restore server

```bash
cat << EOF | docker exec --user=postgres -i pgrestore bash
pg_ctlcluster 13 main stop
rm -rf /var/lib/postgresql/13/main/*
EOF
```


## Check current data

In Runbook 10, we have prepared some data to help us demonstrate a PITR. Check the primary to see the data:

```bash
psql -c 'SELECT * FROM table_test ORDER BY id'
```

You will see each in this case, conveniently each row holds the timestamp of when it was inserted, for example:

```
postgres@027178f33dae:~$ psql -c 'SELECT * FROM table_test ORDER BY id'
 id |             dt
----+----------------------------
  1 | 2021-08-17 03:14:06.312272
  2 | 2021-08-17 03:14:16.329116
  3 | 2021-08-17 03:14:26.332124
(3 rows)
```


## Cause an accident!!

Imagine someone made a mistake and performed a `DELETE` without a `WHERE`:

```bash
psql -c 'DELETE FROM table_test'
```

As a result, the table is now empty:

```
postgres@027178f33dae:~$ psql -c 'DELETE FROM table_test'
DELETE 3
postgres@027178f33dae:~$ psql -c 'SELECT * FROM table_test ORDER BY id'
 id | dt
----+----
(0 rows)
```


## Run the recover

Let's say we can have a very educated guess of when the `DELETE` happened: around `03:14:20` and `03:14:40`. Let's also imagine we know what the table is supposed to look like right before the delete.

So our initial plan is to go back to `2021-08-17 03:14:20` to see how the table was at that time. When Postgres reaches that time, we want it to pause replication, so we can connect and check how is the database, and that allows us to optionally adjust the target time and proceed to recover some more WAL.

Also, order to go back to `03:14:20`, it's required that we recover a base backup from before that timestamp. The list of backups is currently this:

```
barman@1dc73c3167f9:~$ barman list-backup rb08
rb08 20210817T031638 - Tue Aug 17 03:16:38 2021 - Size: 1.5 GiB - WAL Size: 0 B
rb08 20210817T031157 - Tue Aug 17 03:12:06 2021 - Size: 1.5 GiB - WAL Size: 48.0 MiB
```

Which means we need the backup with ID `20210817T031157`, which finished at `03:12:06`, because the one with ID `20210817T031638` finished at `03:16:38`, which is past our recovery target time.

So the command would be like this:

```bash
barman recover \
  --remote-ssh-command "ssh postgres@pgrestore" \
  --target-time "2021-08-17 03:14:20" \
  --target-action pause \
  rb08 20210817T031157 /var/lib/postgresql/13/main/
```

The output should be like this:

```
barman@1dc73c3167f9:~$ barman recover \
>   --remote-ssh-command "ssh postgres@pgrestore" \
>   --target-time "2021-08-17 03:14:20" \
>   --target-action pause \
>   rb08 20210817T031157 /var/lib/postgresql/13/main/
Starting remote restore for server rb08 using backup 20210817T031157
Destination directory: /var/lib/postgresql/13/main/
Remote command: ssh postgres@pgrestore
Doing PITR. Recovery target time: '2021-08-17 03:14:20+00:00'
Using safe horizon time for smart rsync copy: 2021-08-17 03:11:57.986354+00:00
Copying the base backup.
Generating recovery configuration
Identify dangerous settings in destination directory.

IMPORTANT
These settings have been modified to prevent data losses

postgresql.auto.conf line 3: archive_command = false

WARNING
You are required to review the following options as potentially dangerous

postgresql.conf line 41: data_directory = '/var/lib/postgresql/13/main'		# use data in another directory
postgresql.conf line 43: hba_file = '/etc/postgresql/13/main/pg_hba.conf'	# host-based authentication file
postgresql.conf line 45: ident_file = '/etc/postgresql/13/main/pg_ident.conf'	# ident configuration file
postgresql.conf line 49: external_pid_file = '/var/run/postgresql/13-main.pid'			# write an extra PID file
postgresql.conf line 66: unix_socket_directories = '/var/run/postgresql'	# comma-separated list of directories
postgresql.conf line 102: ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
postgresql.conf line 104: ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
postgresql.conf line 771: include_dir = 'conf.d'			# include files ending in '.conf' from

WARNING: 'get-wal' is in the specified 'recovery_options'.
Before you start up the PostgreSQL server, please review the postgresql.auto.conf file
inside the target directory. Make sure that 'restore_command' can be executed by the PostgreSQL user.

Recovery completed (start time: 2021-08-17 04:17:14.005388, elapsed time: 8 seconds)

Your PostgreSQL server has been successfully prepared for recovery!
```


## Check recovery settings

On pgrestore, note that Barman has filled directory `/var/lib/postgresql/13/main/`:

```
postgres@8c31ddda0b2c:~/13/main$ ls -lh
total 156K
-rw------- 1 postgres postgres    3 Aug 17 01:51 PG_VERSION
-rw------- 1 postgres postgres  242 Aug 17 03:11 backup_label
drwxr-xr-x 3 postgres postgres 4.0K Aug 17 04:17 barman_wal
drwx------ 6 postgres postgres 4.0K Aug 17 01:54 base
drwx------ 2 postgres postgres 4.0K Aug 17 03:12 global
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_commit_ts
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_dynshmem
-rw-r----- 1 postgres postgres 4.9K Aug 17 01:51 pg_hba.conf
-rw-r----- 1 postgres postgres 1.6K Aug 17 01:51 pg_ident.conf
drwx------ 4 postgres postgres 4.0K Aug 17 03:11 pg_logical
drwx------ 4 postgres postgres 4.0K Aug 17 01:51 pg_multixact
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_notify
drwx------ 2 postgres postgres 4.0K Aug 17 02:47 pg_replslot
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_serial
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_snapshots
drwx------ 2 postgres postgres 4.0K Aug 17 02:45 pg_stat
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_stat_tmp
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_subtrans
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_tblspc
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_twophase
drwx------ 2 postgres postgres 4.0K Aug 17 02:50 pg_wal
drwx------ 2 postgres postgres 4.0K Aug 17 01:51 pg_xact
-rw------- 1 postgres postgres  424 Aug 17 04:17 postgresql.auto.conf
-rw------- 1 postgres postgres  162 Aug 17 02:45 postgresql.auto.conf.origin
-rw-r--r-- 1 postgres postgres  28K Aug 17 04:17 postgresql.conf
-rw-r--r-- 1 postgres postgres  28K Aug 17 01:51 postgresql.conf.origin
-rw-r--r-- 1 postgres postgres    0 Aug 17 04:17 recovery.signal
```

Barman has also created a file called `recovery.signal` and has filled recovery settings in `postgresql.auto.conf`:

```
postgres@8c31ddda0b2c:~/13/main$ cat postgresql.auto.conf
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
archive_mode = 'on'
#BARMAN#archive_command = 'barman-wal-archive pgbkp1 rb08 %p'
archive_command = false

# The 'barman-wal-restore' command is provided in the 'barman-cli' package
restore_command = 'barman-wal-restore -P -U barman 1dc73c3167f9 rb08 %f %p'
recovery_target_time = '2021-08-17 03:14:20'
recovery_target_action = 'pause'
```

If it was PostgreSQL 11 or older, the recovery settings would have been written to the file `recovery.conf` instead.

We will need to amend the `restore_command`, replacing the container ID with the container name, so it's properly set to:

```
restore_command = 'barman-wal-restore -P -U barman pgbkp1 rb08 %f %p'
```


## Start Postgres

Now let's start Postgres and let it replay WAL files until the `recovery_target_time` is reached.

```bash
docker exec pgrestore pg_ctlcluster 13 main start
```

Check the log file `/var/log/postgresql/postgresql-13-main.log`, you will see this:

```
2021-08-17 04:25:33.105 UTC [9085] LOG:  starting point-in-time recovery to 2021-08-17 03:14:20+00
2021-08-17 04:25:33.627 UTC [9085] LOG:  restored log file "000000010000000000000052" from archive
2021-08-17 04:25:33.649 UTC [9085] LOG:  redo starts at 0/52000028
2021-08-17 04:25:33.650 UTC [9085] LOG:  consistent recovery state reached at 0/52000138
2021-08-17 04:25:33.651 UTC [9084] LOG:  database system is ready to accept read only connections
2021-08-17 04:25:34.169 UTC [9085] LOG:  restored log file "000000010000000000000053" from archive
2021-08-17 04:25:34.190 UTC [9085] LOG:  recovery stopping before commit of transaction 503, time 2021-08-17 03:14:26.332296+00
2021-08-17 04:25:34.190 UTC [9085] LOG:  pausing at the end of recovery
2021-08-17 04:25:34.190 UTC [9085] HINT:  Execute pg_wal_replay_resume() to promote.
```


## Check recovery status

Let's confirm Postgres is still in recovery, recovery is currently paused, and also check the timestamp of the last replayed transaction:

```sql
SELECT pg_is_in_recovery();
SELECT pg_is_wal_replay_paused();
SELECT pg_last_xact_replay_timestamp();
```

Output should be:

```
postgres=# SELECT pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)

postgres=# SELECT pg_is_wal_replay_paused();
 pg_is_wal_replay_paused
-------------------------
 t
(1 row)

postgres=# SELECT pg_last_xact_replay_timestamp();
 pg_last_xact_replay_timestamp
-------------------------------
 2021-08-17 03:14:16.329258+00
(1 row)
```

Let's also check table data:

```
postgres=# SELECT * FROM table_test ORDER BY id;
 id |             dt
----+----------------------------
  1 | 2021-08-17 03:14:06.312272
  2 | 2021-08-17 03:14:16.329116
(2 rows)
```


## Advance recovery by another 10 seconds

We know there is a missing row, so let's advance recovery.

First we need to stop Postgres:

```bash
docker exec pgrestore pg_ctlcluster 13 main stop
```

Now we amend the file `postgresql.auto.conf`, to change the `recovery_target_time`:

```
recovery_target_time = '2021-08-17 03:14:30'
```

Then we start Postgres again:

```bash
docker exec pgrestore pg_ctlcluster 13 main start
```

From log file we see:

```
2021-08-17 04:39:13.595 UTC [9140] LOG:  starting point-in-time recovery to 2021-08-17 03:14:30+00
2021-08-17 04:39:14.103 UTC [9140] LOG:  restored log file "000000010000000000000052" from archive
2021-08-17 04:39:14.125 UTC [9140] LOG:  redo starts at 0/52000028
2021-08-17 04:39:14.650 UTC [9140] LOG:  restored log file "000000010000000000000053" from archive
2021-08-17 04:39:14.672 UTC [9140] LOG:  consistent recovery state reached at 0/530178D8
2021-08-17 04:39:14.673 UTC [9139] LOG:  database system is ready to accept read only connections
2021-08-17 04:39:15.186 UTC [9140] LOG:  restored log file "000000010000000000000054" from archive
2021-08-17 04:39:15.726 UTC [9140] LOG:  restored log file "000000010000000000000055" from archive
2021-08-17 04:39:16.254 UTC [9140] LOG:  restored log file "000000010000000000000056" from archive
2021-08-17 04:39:16.271 UTC [9140] LOG:  recovery stopping before commit of transaction 504, time 2021-08-17 03:56:28.736069+00
2021-08-17 04:39:16.271 UTC [9140] LOG:  pausing at the end of recovery
2021-08-17 04:39:16.271 UTC [9140] HINT:  Execute pg_wal_replay_resume() to promote.
```


## Check recovery status

```sql
SELECT pg_is_in_recovery();
SELECT pg_is_wal_replay_paused();
SELECT pg_last_xact_replay_timestamp();
SELECT * FROM table_test ORDER BY id;
```

Output:

```
postgres=# SELECT pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)

postgres=# SELECT pg_is_wal_replay_paused();
 pg_is_wal_replay_paused
-------------------------
 t
(1 row)

postgres=# SELECT pg_last_xact_replay_timestamp();
 pg_last_xact_replay_timestamp
-------------------------------
 2021-08-17 03:14:26.332296+00
(1 row)

postgres=# SELECT * FROM table_test ORDER BY id;
 id |             dt
----+----------------------------
  1 | 2021-08-17 03:14:06.312272
  2 | 2021-08-17 03:14:16.329116
  3 | 2021-08-17 03:14:26.332124
(3 rows)
```

And we have our complete table data! Which means PITR is complete.

At this point we could:

- Take a pg_dump of this table and restore it to the current primary; or

- Promote this Postgres instance on the pgrestore server to replace the previous primary.
