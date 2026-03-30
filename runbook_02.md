# Setup PostgreSQL cluster

## Objective

Install PostgreSQL 17 and setup a primary and standby.


## Requirements:

- rb01


## Install and configure PostgreSQL 17

```bash
# Define all target containers in one list
for container in pgnode1 pgnode2 pgnode3 pgrestore pgbkp1 pgbkp2
do
    echo "Processing $container..."

    # Check if the container is one of the backup nodes
    if [[ "$container" == "pgbkp1" || "$container" == "pgbkp2" ]]; then
        echo "Installing PostgreSQL 17 binaries ONLY on $container..."
        docker exec "$container" bash -c "
            dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm && \
            dnf -qy module disable postgresql && \
            dnf install -y --allowerasing postgresql17-server postgresql17-contrib wget curl vim"

    else
        # Full installation and configuration for standard nodes
        echo "Performing full PG17 setup on $container..."

        # 1. Install Packages
        docker exec $container dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
        docker exec $container dnf -qy module disable postgresql
        docker exec $container dnf install -y --allowerasing postgresql17-server postgresql17-contrib wget curl vim

        # 2. Initialize the database
        docker exec --user=postgres $container /usr/pgsql-17/bin/initdb -D /var/lib/pgsql/17/data/

        # 3. Configure Postgres settings
        docker exec $container bash -c "echo \"listen_addresses = '*'\" >> /var/lib/pgsql/17/data/postgresql.conf"
        docker exec $container bash -c "echo 'host all all all scram-sha-256' >> /var/lib/pgsql/17/data/pg_hba.conf"
        docker exec $container bash -c "echo 'host replication all all scram-sha-256' >> /var/lib/pgsql/17/data/pg_hba.conf"
    fi

    echo "Finished processing $container."
    echo "-----------------------------------"
done
```


## Feed the primary with data

```bash
# Start Postgres using pg_ctl instead of systemctl
docker exec --user=postgres pgnode1 /usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data/ start
docker exec --user=postgres pgnode1 psql \
  -c "ALTER USER postgres WITH PASSWORD 'Xoo4eiho'"
docker exec --user=postgres pgnode1 psql \
  -c "CREATE DATABASE testdb"
docker exec --user=postgres pgnode1 bash \
  -c "/usr/pgsql-17/bin/pgbench -i -s 100 testdb"
```


## Creating .pgpass

```bash
for container in pgnode1 pgnode2 pgnode3 pgrestore pgbkp1 pgbkp2
do

cat << EOF | docker exec --user=postgres -i $container bash
echo "*:*:*:postgres:Xoo4eiho" > /var/lib/pgsql/.pgpass
chmod 600 /var/lib/pgsql/.pgpass
EOF

done
```


## Build the standby

```bash
docker exec --user=postgres pgnode2 bash \
  -c "rm -rf /var/lib/pgsql/17/data/*"
docker exec --user=postgres pgnode2 bash \
  -c "/usr/pgsql-17/bin/pg_basebackup -h pgnode1 -p 5432 -D /var/lib/pgsql/17/data -U postgres -c fast -X stream -R -C --slot pgnode2"
# Start the standby using pg_ctl
docker exec --user=postgres pgnode2 /usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data/ start
```

Confirm the standby is attached to the primary by connecting to the primary:

```bash
docker exec -it pgnode1 bash
```

Inside the `pgnode1` container, become `postgres` user:

```bash
su - postgres
```

Then connect to the database:

```
psql
```

Inside the database, check the standby is connected:

```sql
SELECT * FROM pg_stat_replication;
SELECT * FROM pg_replication_slots;
```

## The output should look something like this:

```
postgres=# SELECT * FROM pg_stat_replication;
SELECT * FROM pg_replication_slots;
-[ RECORD 1 ]----+------------------------------
pid              | 937
usesysid         | 10
usename          | postgres
application_name | walreceiver
client_addr      | 172.18.0.5
client_hostname  |
client_port      | 36134
backend_start    | 2026-03-05 18:43:23.755833+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/50000060
write_lsn        | 0/50000060
flush_lsn        | 0/50000060
replay_lsn       | 0/50000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2026-03-05 18:46:02.09571+00

-[ RECORD 1 ]-------+-----------
slot_name           | pgnode2
plugin              |
slot_type           | physical
datoid              |
database            |
temporary           | f
active              | t
active_pid          | 937
xmin                |
catalog_xmin        |
restart_lsn         | 0/50000060
confirmed_flush_lsn |
wal_status          | reserved
safe_wal_size       |
two_phase           | f
inactive_since      |
conflicting         |
invalidation_reason |
failover            | f
synced              | f
```