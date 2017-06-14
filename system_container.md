# [Reference](http://www.projectatomic.io/blog/2016/09/intro-to-system-containers/)
`Container` (Thanks to `Docker` to make it usable and pupular) is a revolutionary technology that is changing the way we ship our products, this makes it easy to move the contained application between environments (dev, test, production, etc.) while retaining full functionality.


We can image that the future is a `Containers` world, all the apps will benifit a lot from containers. We can easily run a app in a container nowadays, but how can we run a system service in a container? Here `System container` comes.

`System container` aim is to run services as containers, composed of several projects: 
- [systemd](https://github.com/systemd/systemd) provides a system and service manager that runs as PID 1 and starts the rest of the system
- [runc](https://github.com/opencontainers/runc) is to run OCI containers
- [Atomic](https://github.com/projectatomic/atomic) is a huge project, only the stuff related `System container` feature mentioned below
  - [Skopeo](https://github.com/projectatomic/skopeo) works with remote images registries (Upstream project: [container image](https://github.com/containers/image))
  - [rpm-ostree](https://github.com/projectatomic/rpm-ostree) uses [OSTree](https://ostree.readthedocs.io/en/latest/) as a base image format
- [atomic-system-containers](https://github.com/projectatomic/atomic-system-containers) Collection of system containers images
---

## Docker system container
First, you need download Atomic Centos from https://wiki.centos.org/zh/SpecialInterestGroup/Atomic/Download, and launch it on your IaaS.
- Build a docker-centos system container image as you need

```
# Download the files to build a system container image
$ git clone https://github.com/projectatomic/atomic-system-containers
$ cd atomic-system-containers/docker-centos

# Build image from source code using Docker
$ docker build -t docker-centos .

# Pull the container image via Atomic from local Docker
$ atomic pull --storage ostree docker:docker-centos:latest

# Check if the contaner image is installer properly
$ atomic images list
   REPOSITORY                                                                       TAG         IMAGE ID       CREATED            VIRTUAL SIZE   TYPE
   docker-centos                                                                    latest      708c4bc2534a   2017-06-13 08:30                  ostree  

# Install the container
$ atomic install --system --system-package=no --name=docker-centos docker-centos
Extracting to /var/lib/containers/atomic/docker-centos.0
systemctl daemon-reload
systemd-tmpfiles --create /etc/tmpfiles.d/docker-centos.conf
systemctl enable docker-centos

# Start the docker system contaienr
$ systemctl start docker-centos

```

- How to login to the contanier just like `docker exec`

```
$ runc exec -t docker-centos /bin/bash
```

## What `Atomic` was doing actually while `pull` and `install`

- atomic pull xxxxx

```
# As you can see the container images are stored and maintained in an ostree repo as a branch
$ ll /ostree/repo/
total 16
-rw-r--r--.   1 root root   32 May  2 04:25 config
drwxr-xr-x.   2 root root    6 May  2 04:25 extensions
drwxr-xr-x. 258 root root 8192 May  3 03:46 objects
drwxr-xr-x.   4 root root   34 May  2 04:25 refs
drwxr-xr-x.   2 root root    6 May  2 04:25 state
drwxr-xr-x.   3 root root   19 Jun 13 21:59 tmp

```
- atomic install xxx

```
# There `/var/lib/containers/atomic/docker-centos` directory generated for `docker-centos` ststem container
$ ll /var/lib/containers/atomic
total 0
lrwxrwxrwx. 1 root root  42 Jun 13 08:49 docker-centos -> /var/lib/containers/atomic/docker-centos.0

# Generate some configuration files for `runc` and `systemd`
$ ll /var/lib/containers/atomic/docker-centos/
total 24
-rw-r--r--.  1 root root 9399 Jun 13 08:49 config.json
-rw-r--r--.  1 root root  698 Jun 13 08:49 docker-centos.service
-rw-r--r--.  1 root root 1553 Jun 13 08:49 info
drwxr-xr-x. 15 root root  236 Jun 13 08:49 rootfs
-rw-r--r--.  1 root root   31 Jun 13 08:49 tmpfiles-docker-centos.conf
```

  - [config.json](https://github.com/opencontainers/runtime-spec/blob/master/config.md) is for OCI runtime specs
  - `docker-centos.service` is service unit configuration for systemd
  - `rootfs` file system for LXC containers
  - `info` some generated info automatically 
  - `tmpfiles-docker-centos.conf` tmpfiles.template for system container
