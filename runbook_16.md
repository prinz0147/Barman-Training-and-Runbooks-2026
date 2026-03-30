# Hook script example

## Objective

Write a `post_backup_script` to mimic a copy of all files to tape or the cloud.


## Requirements:

- rb10


## Create the script on the Barman home

On server pgbkp1, as barman user, create a file called `/var/lib/barman/rb16.sh` with the following contents:

```bash
#!/bin/bash

# --- Modernized Barman Post-Backup Hook ---
# Environment: CentOS 9 / Docker / PG 17
# Objective: Securely log and mimic cloud/tape transfer

set -e # Exit immediately if a command exits with a non-zero status
set -u # Treat unset variables as an error

# Ensure required Barman variables are present
: "${BARMAN_SERVER:?Variable BARMAN_SERVER not set}"
: "${BARMAN_BACKUP_ID:?Variable BARMAN_BACKUP_ID not set}"

LOGFILE="$HOME/hook_${BARMAN_SERVER}_${BARMAN_BACKUP_ID}.log"

echo "[$(date +'%Y-%m-%d %H:%M:%S')] Starting post-backup transfer for ${BARMAN_SERVER}" >> "$LOGFILE"

# Use 'while read' to safely handle filenames and avoid subshell overhead
barman list-files "$BARMAN_SERVER" "$BARMAN_BACKUP_ID" | while read -r f
do
  # Using printf is more portable and predictable than echo -n
  printf "Copying file %s ... " "$f" >> "$LOGFILE"

  # --- ACTUAL TRANSFER COMMAND GOES HERE ---
  # Example for S3: aws s3 cp "$f" s3://my-bucket/backup/
  # Example for Tape/Local: cp "$f" /mnt/archive/
  sleep 0.01

  echo "Done." >> "$LOGFILE"
done

echo "[$(date +'%Y-%m-%d %H:%M:%S')] Transfer completed successfully." >> "$LOGFILE"
```

Give the file execution permissions:

```bash
chmod +x rb16.sh
```

As root user, amend file `/etc/barman.d/rb08.conf`, add the following line:

```
post_backup_script = "/var/lib/barman/rb16.sh"
```
## To verify:

```bash
[barman@1c37152b8827 ~]$ ll
total 8
-rw-r--r--. 1 barman barman    2 Mar 17 17:09 cfg_changes.queue
drwxr-xr-x. 8 barman barman  130 Mar 17 16:44 rb08
drwxr-xr-x. 7 barman barman   77 Mar 11 00:31 rb09
-rwxr-xr-x. 1 barman barman 1153 Mar 17 17:07 rb16.sh
```

Then Run `barman cron`.

```bash
[barman@1c37152b8827 ~]$ barman cron
Starting WAL archiving for server rb08
```


## Test the hook script

In order to test the hook script, run a `barman backup`:

```bash
barman backup rb08
```

Note how it takes a little longer. While the script is running, the backup status is `WAITING_FOR_WALS`.

Also check the home directory, a log file should be created there.

## The output will look like this:

```bash
[barman@1c37152b8827 ~]$ barman backup rb08
Starting backup using rsync-concurrent method for server rb08 in /var/lib/barman/rb08/base/20260317T170957
Backup start at LSN: 0/83000028 (000000010000000000000083, 00000028)
Starting backup copy via rsync/SSH for 20260317T170957 (4 jobs)
Copy done (time: less than one second)
Asking PostgreSQL server to finalize the backup.
Backup size: 1.5 GiB. Actual size on disk: 8.2 KiB (-100.00% deduplication ratio).
Backup end at LSN: 0/83000120 (000000010000000000000083, 00000120)
Backup completed (start time: 2026-03-17 17:09:57.367255, elapsed time: 7 seconds)
Processing xlog segments from streaming for rb08 (batch size: 3)
        000000010000000000000082
        000000010000000000000083
        000000010000000000000084
Processing xlog segments from file archival for rb08
        000000010000000000000082
        000000010000000000000083
        000000010000000000000083.00000028.backup
        000000010000000000000084
[barman@1c37152b8827 ~]$
[barman@1c37152b8827 ~]$
[barman@1c37152b8827 ~]$
[barman@1c37152b8827 ~]$
[barman@1c37152b8827 ~]$
[barman@1c37152b8827 ~]$ barman list-backups rb08
rb08 20260317T170957 - R - Tue Mar 17 17:09:59 2026 - Size: 1.5 GiB - WAL Size: 16.0 MiB
rb08 20260317T165956 - R - Tue Mar 17 16:59:58 2026 - Size: 1.5 GiB - WAL Size: 48.0 MiB
rb08 20260311T174245 - R - Wed Mar 11 17:42:47 2026 - Size: 1.5 GiB - WAL Size: 176.0 MiB
rb08 20260311T154944 - R - Wed Mar 11 15:49:46 2026 - Size: 1.5 GiB - WAL Size: 96.0 MiB
rb08 20260311T154518 - R - Wed Mar 11 15:45:20 2026 - Size: 1.5 GiB - WAL Size: 48.0 MiB
rb08 20260311T150418 - R - Wed Mar 11 15:04:20 2026 - Size: 1.5 GiB - WAL Size: 128.0 MiB
[barman@1c37152b8827 ~]$ vi rb16.sh
[barman@1c37152b8827 ~]$ ll
total 200
-rw-r--r--. 1 barman barman      2 Mar 17 17:09 cfg_changes.queue
-rw-r--r--. 1 barman barman 107806 Mar 17 17:10 hook_rb08_20260317T170957.log --------------------> NOTICE THE LOG CREATED HERE
drwxr-xr-x. 8 barman barman    130 Mar 17 16:44 rb08
drwxr-xr-x. 7 barman barman     77 Mar 11 00:31 rb09
-rwxr-xr-x. 1 barman barman   1153 Mar 17 17:07 rb16.sh
```
