# Install Docker

## Objective

Install Docker on the machine (physical or virtual) that the attendees will use to run the experiments and study.


## Requirements:

- None

## Install Docker

Reference: https://docs.docker.com/engine/install/centos/ AND https://docs.docker.com/engine/install/debian/

## Uninstall Old versions of Docker:

```
sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

## Setup the repository:

```
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

## On Debian 10, as `root`:

```
apt update
apt -y upgrade
apt install ca-certificates curl gnupg2 
curl -fsSL https://download.docker.com/linux/debian/gpg | dnf-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian buster stable"
apt update
apt install docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $(whoami)
```

## Install the Latest Version:

```
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```


## Start Docker Engine

```
sudo systemctl enable --now docker
```

# Verify that docker is successfully installed by running the `hello-world` image:

```
sudo docker run hello-world
```

# And you should see something like the below example:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
