# Setup Docker containers

## Objective

Create a Docker network and 6 Docker containers based on CentOS 9.


## Requirements:

- rb00

- Docker installed and running.


## Setup

Use the script below to create a Docker network and 6 Docker containers. This version uses `quay.io/centos/centos:stream9` and installs essential tools (`coreutils`, `util-linux`, `hostname`, `iputils`, and `procps-ng`) to ensure commands like `ls`, `ll`, and `hostname` work out of the box.

```bash
docker network create barman_network 2>/dev/null || true

# Define the list of containers
containers=("pgbkp1" "pgbkp2" "pgnode1" "pgnode2" "pgnode3" "pgrestore")

# Loop to create each container with CentOS Stream 9
for name in "${containers[@]}"; do
    echo "--- Initializing container: $name ---"
    
    # Run container
    docker run -itd --network=barman_network --name="$name" quay.io/centos/centos:stream9 bash

    # 1. Base System & Requested Utilities
    # findutils provides 'find', util-linux provides 'whereis'
    docker exec "$name" bash -c "
        dnf clean all && \
        dnf -y install --allowerasing \
        coreutils util-linux hostname iputils procps-ng \
        rsync findutils which bash-completion iproute \
        net-tools tar gzip shadow-utils ncurses \
        passwd sudo openssh-server openssh-clients dnf-plugins-core && \
        dnf -y groupinstall 'Development Tools' && \
        echo 'alias ll=\"ls -la --color=auto\"' >> ~/.bashrc"

    echo "Finished configuring $name."
done
```


## Connect

You can connect to each container for example:

```bash
docker exec -it pgnode1 bash
```


## Tear Down

Once you have finished your tests with this cluster, you can destroy it with:

```bash
docker rm -f pgbkp1
docker rm -f pgbkp2
docker rm -f pgnode1
docker rm -f pgnode2
docker rm -f pgnode3
docker rm -f pgrestore
docker network rm barman_network
```
