# Install Barman packages

## Objective

Install `barman` package on Barman servers and `barman-cli` package on all servers.


## Requirements:

- rb02
- Containers: `pgbkp1, pgbkp2, pgnode1, pgnode2, pgnode3, pgrestore`


## Install barman package

```bash
for container in pgbkp1 pgbkp2
do

docker exec $container dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
docker exec $container dnf install -y epel-release
docker exec $container dnf update -y
docker exec $container dnf install -y wget curl vim
docker exec $container dnf install -y barman

done
```


## Install barman-cli package

```bash
for container in pgbkp1 pgbkp2 pgnode1 pgnode2 pgnode3 pgrestore
do
docker exec $container dnf install -y barman-cli
done
```
