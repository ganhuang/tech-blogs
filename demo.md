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

## atomic

```
systemctl stop docker
atomic pull --storage=ostree  docker.io/ghuang/hello-world
atomic install --system-package=no docker.io/ghuang/hello-world
http://10.x.x.x:8081/
```
