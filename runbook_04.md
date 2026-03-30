# Barman prerequisites

## Objective

Performs the following actions:

- Install SSH server
- Exchange SSH keys
- Create Barman database users
- Setup .pgpass for Barman


## Requirements:

- rb03


## Install and start cron

```bash
for container in pgbkp1 pgbkp2 pgnode1 pgnode2 pgnode3 pgrestore
do
docker exec $container dnf install -y cronie
docker exec $container bash -c "/usr/sbin/crond"
done
```


## Install and start SSH server

```bash
#!/bin/bash

# Define the list of containers
containers=("pgbkp1" "pgbkp2" "pgnode1" "pgnode2" "pgnode3" "pgrestore")

for container in "${containers[@]}"
do
    echo "Configuring SSH on: $container..."

    # 1. Install packages
    docker exec "$container" dnf install -y openssh-server openssh-clients

    # 2. Generate host keys (only if they don't exist)
    docker exec "$container" ssh-keygen -A

    # 3. Set a root password (Optional but often necessary for SSH access)
    # docker exec "$container" bash -c "echo 'root:password' | chpasswd"

    # 4. Start the SSH daemon in the background
    # We use the full path and ensure it doesn't detach in a way that kills the process
    docker exec -d "$container" /usr/sbin/sshd -D
    
    echo "Done with $container."
done
```


## Exchange SSH keys

```bash
#!/bin/bash

# Define home paths explicitly
BARMAN_HOME="/var/lib/barman"
POSTGRES_HOME="/var/lib/pgsql"

echo "Cleaning existing SSH configurations..."
# 1. Clean existing keys
for container in pgbkp1 pgbkp2; do 
    docker exec -u barman $container bash -c "rm -rf $BARMAN_HOME/.ssh"
done

for container in pgnode1 pgnode2 pgnode3 pgrestore; do 
    docker exec -u postgres $container bash -c "rm -rf $POSTGRES_HOME/.ssh"
done

echo "Generating master key on pgbkp1..."
# 2. Generate pgbkp1 key
docker exec -u barman pgbkp1 mkdir -p $BARMAN_HOME/.ssh
docker exec -u barman pgbkp1 chmod 700 $BARMAN_HOME/.ssh
docker exec -u barman pgbkp1 ssh-keygen -t rsa -b 2048 -f $BARMAN_HOME/.ssh/id_rsa -N "" -q

# Disable StrictHostKeyChecking for seamless automation
docker exec -u barman pgbkp1 bash -c "echo -e 'Host *\n  StrictHostKeyChecking no\n  UserKnownHostsFile /dev/null' > $BARMAN_HOME/.ssh/config"
docker exec -u barman pgbkp1 chmod 600 $BARMAN_HOME/.ssh/config

echo "Distributing keys to nodes..."
# 3. Extract the newly created keys to the host temporarily
docker cp pgbkp1:$BARMAN_HOME/.ssh/id_rsa ./id_rsa_tmp
docker cp pgbkp1:$BARMAN_HOME/.ssh/id_rsa.pub ./id_rsa_tmp.pub

# 4. Distribute to all other containers
for container in pgnode1 pgnode2 pgnode3 pgrestore pgbkp2; do
    if [[ "$container" == "pgbkp2" ]]; then
        DEST=$BARMAN_HOME/.ssh
        U_NAME="barman"
    else
        DEST=$POSTGRES_HOME/.ssh
        U_NAME="postgres"
    fi

    # Create directory and copy files
    docker exec $container mkdir -p $DEST
    docker cp ./id_rsa_tmp $container:$DEST/id_rsa
    docker cp ./id_rsa_tmp.pub $container:$DEST/id_rsa.pub
    docker cp ./id_rsa_tmp.pub $container:$DEST/authorized_keys

    # Fix ownership and permissions inside the container
    docker exec $container chown -R $U_NAME:$U_NAME $DEST
    docker exec $container chmod 700 $DEST
    docker exec $container chmod 600 $DEST/id_rsa $DEST/authorized_keys
    
    # Also disable host checking on the backup replica (pgbkp2)
    if [[ "$container" == "pgbkp2" ]]; then
        docker exec -u barman pgbkp2 bash -c "echo -e 'Host *\n  StrictHostKeyChecking no\n  UserKnownHostsFile /dev/null' > $DEST/config"
        docker exec $container chmod 600 $DEST/config
    fi
done

# Clean up host temporary files
rm -f id_rsa_tmp id_rsa_tmp.pub

echo "---------------------------------------"
echo "SSH Key Exchange Complete."
echo "Testing connection from pgbkp1 to pgnode1..."
docker exec -u barman pgbkp1 ssh postgres@pgnode1 "echo 'SSH connection successful!'"
```

# Fix Compatibility Issues

```bash
for container in pgbkp1 pgbkp2
do
docker exec $container ln -sf /usr/bin/pg_receivewal /usr/bin/pg_receivexlog
done
```


## Create Barman database users

```bash
cat << EOF | docker exec --user=postgres -i pgnode1 psql
CREATE USER barman WITH SUPERUSER PASSWORD 'eilo4Eim';
CREATE USER streaming_barman WITH REPLICATION PASSWORD 'ye1Thie5';
EOF
```


## Creating .pgpass for Barman

```bash
#!/bin/bash

# Define the containers to target
CONTAINERS=("pgbkp1" "pgbkp2")

for container in "${CONTAINERS[@]}"
do
    echo "--- Updating .pgpass in $container ---"

    # Using <<'EOF' (quoted) prevents local shell expansion, 
    # ensuring all variables are evaluated inside the container.
    docker exec --user=root -i "$container" bash <<'EOF'
        # 1. Dynamically find the barman home directory
        BARMAN_HOME=$(getent passwd barman | cut -d: -f6)

        # Safety check: Exit if barman user doesn't exist
        if [ -z "$BARMAN_HOME" ]; then
            echo "Error: User 'barman' not found in $(hostname)"
            exit 1
        fi

        PGPASS_PATH="$BARMAN_HOME/.pgpass"

        # 2. Write credentials to the file
        # We use a temp file or direct redirection to ensure clean content
        cat <<INNER_EOF > "$PGPASS_PATH"
*:*:*:barman:eilo4Eim
*:*:*:streaming_barman:ye1Thie5
INNER_EOF

        # 3. Set strict permissions (Postgres ignores .pgpass if it's too open)
        chown barman:barman "$PGPASS_PATH"
        chmod 0600 "$PGPASS_PATH"

        echo "Successfully updated .pgpass at $PGPASS_PATH"
EOF
done
```
