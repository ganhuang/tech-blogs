## buildah

```
systemctl stop docker
buildah bud . -t hello-world
buildah images
buildah tag docker.io/library/hello-world:latest  ghuang/hello-world
docker login docker.io
kpod login docker.io
buildah push docker.io/ghuang/hello-world:latest  docker://docker.io/ghuang/hello-world:latest
```

## skopeo

```
systemctl stop docker
skopeo inspect docker://docker.io/ghuang/hello-world
skopeo inspect docker://registry.access.redhat.com/openshift3/ose
skopeo copy docker://docker.io/busybox docker://docker.io/ghuang/busybox
```

## atomic

```
systemctl stop docker
atomic pull --storage=ostree  docker.io/ghuang/hello-world
atomic install --system-package=no docker.io/ghuang/hello-world
```

```
# systemctl cat hello-world
[Service]
ExecStart=/bin/runc --systemd-cgroup run 'hello-world'
ExecStop=/bin/runc --systemd-cgroup kill 'hello-world'
Restart=on-failure
WorkingDirectory=/var/lib/containers/atomic/hello-world.0
```

```
curl http://10.x.x.x:8081/
```
