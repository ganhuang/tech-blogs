# [Reference](http://www.projectatomic.io/blog/2016/09/intro-to-system-containers/)
`Container` (Thanks to `Docker` to make it usable and pupular) is a revolutionary technology that is changing the way we ship our products, making them being iterated faster as well.


We can image that the future is a `Containers` world, all the apps will benifit a lot from containers. We can easily run a app in a container nowadays, but how can we run a system service in a container? `System container` comes.

`System container` aim is to run services as containers, composed of several projects: 
- [systemd](https://github.com/systemd/systemd) provides a system and service manager that runs as PID 1 and starts the rest of the system
- [runc](https://github.com/opencontainers/runc) is to run OCI containers
- [Atomic](https://github.com/projectatomic/atomic) is a huge project, only the stuff related `System container` feature mentioned below
  - [Skopeo](https://github.com/projectatomic/skopeo) works with remote images registries (Upstream project: [container-image](https://github.com/containers/image))
  - [rpm-ostree](https://github.com/projectatomic/rpm-ostree) uses [OSTree](https://ostree.readthedocs.io/en/latest/) as a base image format
- [atomic-system-containers](https://github.com/projectatomic/atomic-system-containers) Collection of system containers images
---

## Docker system container

