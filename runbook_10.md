# Backup

## Objective

Perform initial backups as part of server setup. Understand minimum redundancy requirements and deduplication.


## Requirements:

- rb05, rb06, rb08, rb08 or rb09


## Run a backup

```bash
barman backup rb08
```

Output will be similar to:

```
barman@9c99447bf70f:~$ barman backup rb08
Starting backup using rsync-exclusive method for server rb08 in /var/lib/barman/rb08/base/20210810T044727
Backup start at LSN: 0/52000028 (000000010000000000000052, 00000028)
This is the first backup for server rb08
WAL segments preceding the current backup have been found:
	000000010000000000000050 from server rb08 has been removed
Starting backup copy via rsync/SSH for 20210810T044727 (4 jobs)
rb08: pg_receivewal: finished segment at 0/52000000 (timeline 1)
Copy done (time: 7 seconds)
This is the first backup for server rb08
Asking PostgreSQL server to finalize the backup.
rb08: pg_receivewal: finished segment at 0/53000000 (timeline 1)
Backup size: 1.5 GiB. Actual size on disk: 1.5 GiB (-0.00% deduplication ratio).
Backup end at LSN: 0/52000138 (000000010000000000000052, 00000138)
Backup completed (start time: 2021-08-10 04:47:27.244705, elapsed time: 10 seconds)
Processing xlog segments from streaming for rb08
	000000010000000000000050
	000000010000000000000051
	000000010000000000000052
Processing xlog segments from file archival for rb08
	000000010000000000000051
	000000010000000000000052
	000000010000000000000052.00000028.backup
```

List backups with:

```bash
barman list-backup rb08
```

Output will be like:

```
barman@9c99447bf70f:~$ barman list-backup rb08
rb08 20210810T044727 - Tue Aug 10 04:47:35 2021 - Size: 1.5 GiB - WAL Size: 0 B
```

If you want to list backups for all servers, use:

```bash
barman list-backup all
```

You can see details of the backups with:

```bash
barman show-backup rb08 20210810T044727
```

Being `rb08` the server name, and `20210810T044727` the backup ID.

Output:

```
barman@9c99447bf70f:~$ barman show-backup rb08 20210810T044727
Backup 20210810T044727:
  Server Name            : rb08
  System Id              : 6994591140492671773
  Status                 : DONE
  PostgreSQL Version     : 130003
  PGDATA directory       : /var/lib/postgresql/13/main

  Base backup information:
    Disk usage           : 1.5 GiB (1.5 GiB with WALs)
    Incremental size     : 1.5 GiB (-0.00%)
    Timeline             : 1
    Begin WAL            : 000000010000000000000052
    End WAL              : 000000010000000000000052
    WAL number           : 1
    Begin time           : 2021-08-10 04:47:27.193307+00:00
    End time             : 2021-08-10 04:47:35.130864+00:00
    Copy time            : 7 seconds
    Estimated throughput : 208.4 MiB/s (4 jobs)
    Begin Offset         : 40
    End Offset           : 312
    Begin LSN           : 0/52000028
    End LSN             : 0/52000138

  WAL information:
    No of files          : 0
    Disk usage           : 0 B
    Last available       : 000000010000000000000052

  Catalog information:
    Retention Policy     : VALID
    Previous Backup      : - (this is the oldest base backup)
    Next Backup          : - (this is the latest base backup)
```


## Minimum redundancy requirements

Run `barman check rb08`, output will be:

```
barman@9c99447bf70f:~$ barman check all
Server rb08:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: OK (interval provided: 2 days, latest backup age: 14 minutes, 54 seconds)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: FAILED (have 1 backups, expected at least 2)
	ssh: OK (PostgreSQL server)
	not in recovery: OK
	systemid coherence: OK
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archive_mode: OK
	archive_command: OK
	continuous archiving: OK
	archiver errors: OK
```

We are still not all green: minimum redundancy requirements is still failing. We need a second backup.

But first, let's add some changes to the Postgres database, as if we were using it in the real world. These changes will help us test PITR later.

```bash
cat << EOF | docker exec --user=postgres -i pgnode1 psql
CREATE TABLE table_test (
  id INTEGER PRIMARY KEY,
  dt TIMESTAMP WITHOUT TIME ZONE DEFAULT now()
);
INSERT INTO table_test VALUES (1);
SELECT pg_sleep(10);
INSERT INTO table_test VALUES (2);
SELECT pg_sleep(10);
INSERT INTO table_test VALUES (3);
SELECT pg_switch_wal();
SELECT pg_switch_wal();
EOF
```

On Barman, make sure `barman cron` has executed:

```bash
barman cron
```

Then list the backups, note how the backup has 1 WAL file:

```
barman@9c99447bf70f:~$ barman list-backup all
rb08 20210810T044727 - Tue Aug 10 04:47:35 2021 - Size: 1.5 GiB - WAL Size: 16.0 MiB
```

Then take another backup and list backups again:

```bash
barman backup rb08
barman list-backup rb08
```

Output:

```
barman@9c99447bf70f:~$ barman list-backup all
rb08 20210810T052316 - Tue Aug 10 05:23:16 2021 - Size: 1.5 GiB - WAL Size: 0 B
rb08 20210810T044727 - Tue Aug 10 04:47:35 2021 - Size: 1.5 GiB - WAL Size: 48.0 MiB
```


## Checking deduplication

The second backup is much smaller because it contains only the files that have changed. All other files are hard links to the files of the previous backup.

```
barman@9c99447bf70f:~$ barman show-backup rb08 latest
Backup 20210810T052316:
  Server Name            : rb08
  System Id              : 6994591140492671773
  Status                 : DONE
  PostgreSQL Version     : 130003
  PGDATA directory       : /var/lib/postgresql/13/main

  Base backup information:
    Disk usage           : 1.5 GiB (1.5 GiB with WALs)
    Incremental size     : 2.4 MiB (-99.85%)
    Timeline             : 1
    Begin WAL            : 000000010000000000000055
    End WAL              : 000000010000000000000055
    WAL number           : 1
    Begin time           : 2021-08-10 05:23:16.033934+00:00
    End time             : 2021-08-10 05:23:16.770239+00:00
    Copy time            : less than one second
    Estimated throughput : 6.1 MiB/s (4 jobs)
    Begin Offset         : 40
    End Offset           : 312
    Begin LSN           : 0/55000028
    End LSN             : 0/55000138

  WAL information:
    No of files          : 6
    Disk usage           : 96.0 MiB
    WAL rate             : 0.04/hour
    Last available       : 00000001000000000000005B

  Catalog information:
    Retention Policy     : VALID
    Previous Backup      : 20210810T044727
    Next Backup          : - (this is the latest base backup)
```

In this case it has a deduplication rate of 99.85%. The base backup is still 1.5GB of size, but on disk only 2.4MB are used.

We can confirm that in another way:

```
barman@9c99447bf70f:~$ cd rb08/base/
barman@9c99447bf70f:~/rb08/base$ du -sh --apparent-size *
1.5G	20210810T044727
2.5M	20210810T052316
```

The highest the deduplication rate, the smallest the backup will be on disk, and the fastest the `barman backup` command will be.


## Delete a backup

Barman automatically deletes the oldest backup that is not within the retention policy. But we can also manually delete the oldest backup with:

```
barman delete rb08 oldest
```

If you try the above command, you will see the following error:

```
barman@9c99447bf70f:~/rb08/base$ barman delete rb08 oldest
WARNING: Skipping delete of backup 20210810T044727 for server rb08 due to minimum redundancy requirements (minimum redundancy = 2, current redundancy = 2)
...
```

Which means Barman prevents us from deleting the oldest backup because of `minimum_redundancy`.

If we create a third new backup, it would be possible to delete the oldest backup.

Remember: Instead of `oldest`, we can specify any backup ID in `barman delete`.

For example, let's create a new backup and then delete it:

```
barman@9c99447bf70f:~$ barman backup rb08
Starting backup using rsync-exclusive method for server rb08 in /var/lib/barman/rb08/base/20210816T202451
Backup start at LSN: 0/5D000028 (00000001000000000000005D, 00000028)
Starting backup copy via rsync/SSH for 20210816T202451 (4 jobs)
Copy done (time: less than one second)
Asking PostgreSQL server to finalize the backup.
Backup size: 1.5 GiB. Actual size on disk: 2.2 MiB (-99.86% deduplication ratio).
Backup end at LSN: 0/5D000100 (00000001000000000000005D, 00000100)
Backup completed (start time: 2021-08-16 20:24:51.155925, elapsed time: 1 second)
Processing xlog segments from streaming for rb08
	00000001000000000000005C
	00000001000000000000005D
Processing xlog segments from file archival for rb08
	00000001000000000000005C
	00000001000000000000005D
	00000001000000000000005D.00000028.backup

barman@9c99447bf70f:~$ barman list-backup rb08
rb08 20210816T202451 - Mon Aug 16 20:24:51 2021 - Size: 1.5 GiB - WAL Size: 0 B
rb08 20210810T052316 - Tue Aug 10 05:23:16 2021 - Size: 1.5 GiB - WAL Size: 128.0 MiB
rb08 20210810T044727 - Tue Aug 10 04:47:35 2021 - Size: 1.5 GiB - WAL Size: 48.0 MiB

barman@9c99447bf70f:~$ barman delete rb08 latest
Deleting backup 20210816T202451 for server rb08
Deleted backup 20210816T202451 (start time: Mon Aug 16 20:25:29 2021, elapsed time: less than one second)

barman@9c99447bf70f:~$ barman list-backup rb08
rb08 20210810T052316 - Tue Aug 10 05:23:16 2021 - Size: 1.5 GiB - WAL Size: 128.0 MiB
rb08 20210810T044727 - Tue Aug 10 04:47:35 2021 - Size: 1.5 GiB - WAL Size: 48.0 MiB
```
