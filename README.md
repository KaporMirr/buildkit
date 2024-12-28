[![asciicinema example](https://.org/a/gPEIEo1NzmDTUu2bEPsUboqmU.png)](https://asciinema.org/a/gPEIEo1NzmDTUu2bEPsUboqmU)

# BuildKit

[![GoDoc](https://godoc.org/github.com/moby/buildkit?status.svg)](https://godoc.org/github.com/moby/buildkit/client/llb)
[![Build Status](https://travis-ci.org/moby/buildkit.svg?branch=master)](
BuildKit is a toolkit for converting source code to build artifacts in an efficient, expressive and repeatable manner.

Key features:

-   Automatic garbage collection
-   Extendable frontend formats
-   Concurrent dependency resolution
-   Efficient instruction caching
-   Build cache import/export
-   Nested build job invocations
-   Distributable workers
-   Multiple output formats
-   Pluggable architecture
-   Execution without root privileges

Read the proposal from https://github.com/moby/moby/issue
ile/docs/experi](frontend/dockerfile/docs/experimental.md).

:information_source: [BuildKit has been integrated to `docker build` since

<!-- END doctoc generated TOC please keep comment here to allow auto update Used by

BuildKit is used by the following proje
-   [img](https://github.com/genuinetools/img)
-   [OpenFaaS Cloud](https://github.com/openfaas/openfaas-clo
-   [Tekton Pipelines](https://github.com/tektoncd/catalog) (formerly [Knative Build Templates](https://github.com/knative/build-templates))
-   [the Sanic build tool](https://github.com/distributed-containers-inc/sanic)
-   [vab](https://github.com/stellarproject/vab)
-   [Rio](https://github.com/rancher/rio)
-   [PouchContainer](https://github.com/alibaba/pouch)
-   [Docker buildx](https://github.com/docker/buildx)
-   [Okteto Cloud](https://okteto.com/)

## Quick start

:information_source: For Kubernetes deployments, see [`examples/kubernetes`](./examples/kubernetSee [Expose BuildKit](#expose-buildkit-as-a-tcp-service).

:information_source: Notice to Fedora 31 users:

* As runc still does not work on cgroup like Fedora 31, you need to substitute runc with crun. Run `buildkitd`  `--oci-work

-   Marshaled as Protobuf messages
-   Concurrently 

-   Dockerfile (See [Exploring Dockerfiles](#exploring-

-   [Gockerfi)
-   (open a PR to add your own language)

### Exploring Dockerfiles

Frontends are components that run inside BuildKit and convert any build definition to LLB. There is a special frontend called gateway (`gateway.v0`) that allows using any image as a frontend.

During development, Dockerfile frontend (`dockerfile.v0`) is also part of the BuildKit repo. In the future, this will be moved out, and Dockerfiles can be built using an external image.

```
    --=dockerfile.v0 \
    --local context



```
Details in [Export cache](#export-cache).

```bash
buildctl build ...\
  --output type=image,name=docker.io/username/image,push=true \
  --export-cache type=inline \
  --import-cache type=registry,ref=docker.io/username/image
```

Keys supported by image output:
* `name=[value]`: image name
* `push=true`: push after creating the image
* `push-by-digest=true`: push unnamed image
* `registry.insecure=true`: push to insecure HTTP registry
* `oci-mediatypes=true`: use OCI mediatypes in configuration JSON instead of Docker's
* `unpack=true`: unpack image after creation (for use with containerd)
* `dangling-name-prefix=[value]`: name image with `prefix@<digest>` , used for anonymous images
* `name-canonical=true`: add additional canonical name `name@<digest>`
* `compression=[uncompressed,gzip]`: choose compression type for layer, gzip is default value


If credentials are required, `buildctl` will attempt to read Docker configuration file `$DOCKER_CONFIG/config.json`.
`$DOCKER_CONFIG` defaults to `~/.docker`.

#### Local directory

The local client will copy the files directly to the client. This is useful if BuildKit is being used for building something else than container images.

```bash
buildctl build ... --output type=local,dest=path/to/output-dir
```

To export specific files use multi-stage builds with a scratch stage and copy the needed files into that stage with `COPY --from`.

```dockerfile
...
FROM scratch as testresult

COPY --from=builder /usr/src/app/testresult.xml .
...
```

```bash
buildctl build ... --opt target=testresult --output type=local,dest=path/to/output-dir
```

Tar exporter is similar to local exporter but transfers the files through a tarball.

```bash
buildctl build ... --output type=tar,dest=out.tar
buildctl build ... --output type=tar > out.tar
```

#### Docker tarball

```bash
# exported tarball is also compatible with OCI spec
buildctl build ... --output type=docker,name=myimage | docker load
```

#### OCI tarball

```bash
buildctl build ... --output type=oci,dest=path/to/output.tar
buildctl build ... --output type=oci > output.tar
```
#### containerd image store

The containerd worker needs to be used

```bash
buildctl build ... --output type=image,name=docker.io/username/image
ctr --namespace=buildkit images ls
```

To change the containerd namespace, you need to change `worker.containerd.namespace` in [`/etc/buildkit/buildkitd.toml`](./docs/buildkitd.toml.md).


## Cache

To show local build cache (`/var/lib/buildkit`):

```bash
buildctl du -v
```

To prune local build cache:
```bash
buildctl prune
```

### Garbage collection

See [`./docs/buildkitd.toml.md`](./docs/buildkitd.toml.md).

### Export cache

BuildKit supports the following cache exporters:
* `inline`: embed the cache into the image, and push them to the registry together
* `registry`: push the image and the cache separately
* `local`: export to a local directory

In most case you want to use the `inline` cache exporter.
However, note that the `inline` cache exporter only supports `min` cache mode. 
To enable `max` cache mode, push the image and the cache separately by using `registry` cache exporter.

#### Inline (push image and cache together)

```bash
buildctl build ... \
  --output type=image,name=docker.io/username/image,push=true \
  --export-cache type=inline \
  --import-cache type=registry,ref=docker.io/username/image
```

Note that the inline cache is not imported unless `--import-cache type=registry,ref=...` is provided.

:information_source: Docker-integrated BuildKit (`DOCKER_BUILDKIT=1 docker build`) and `docker buildx`requires 
`--build-arg BUILDKIT_INLINE_CACHE=1` to be specified to enable the `inline` cache exporter.
However, the standalone `buildctl` does NOT require `--opt build-arg:BUILDKIT_INLINE_CACHE=1` and the build-arg is simply ignored.

#### Registry (push image and cache separately)

```bash
buildctl build ... \
  --output type=image,name=localhost:5000/myrepo:image,push=true \
  --export-cache type=registry,ref=localhost:5000/myrepo:buildcache \
  --import-cache type=registry,ref=localhost:5000/myrepo:buildcache \
```

#### Local directory

```bash
buildctl build ... --export-cache type=local,dest=path/to/output-dir
buildctl build ... --import-cache type=local,src=path/to/input-dir
```

The directory layout conforms to OCI Image Spec v1.0.

#### `--export-cache` options
-   `type`: `inline`, `registry`, or `local`
-   `mode=min` (default): only export layers for the resulting image
-   `mode=max`: export all the layers of all intermediate steps. Not supported for `inline` cache exporter.
-   `ref=docker.io/user/image:tag`: reference for `registry` cache exporter
-   `dest=path/to/output-dir`: directory for `local` cache exporter

#### `--import-cache` options
-   `type`: `registry` or `local`. Use `registry` to import `inline` cache.
-   `ref=docker.io/user/image:tag`: reference for `registry` cache importer
-   `src=path/to/input-dir`: directory for `local` cache importer
-   `digest=sha256:deadbeef`: digest of the manifest list to import for `local` cache importer.
-   `tag=customtag`: custom tag of image for `local` cache importer.
    Defaults to the digest of "latest" tag in `index.json` is for digest, not for tag

### Consistent hashing

If you have multiple BuildKit daemon instances but you don't want to use registry for sharing cache across the cluster,
consider client-side load balancing using consistent hashing.

See [`./examples/kubernetes/consistenthash`](./examples/kubernetes/consistenthash).

## Expose BuildKit as a TCP service

The `buildkitd` daemon can listen the gRPC API on a TCP socket.

It is highly recommended to create TLS certificates for both the daemon and the client (mTLS).
Enabling TCP without mTLS is dangerous because the executor containers (aka Dockerfile `RUN` containers) can call BuildKit API as well.

```bash
buildkitd \
  --addr tcp://0.0.0.0:1234 \
  --tlscacert /path/to/ca.pem \
  --tlscert /path/to/cert.pem \
  --tlskey /path/to/key.pem
```

```bash
buildctl \
  --addr tcp://example.com:1234 \
  --tlscacert /path/to/ca.pem \
  --tlscert /path/to/clientcert.pem \
  --tlskey /path/to/clientkey.pem \
  build ...
```

### Load balancing

`buildctl build` can be called against randomly load balanced the `buildkitd` daemon.

See also [Consistent hashing](#consistenthashing) for client-side load balancing.

## Containerizing BuildKit

BuildKit can also be used by running the `buildkitd` daemon inside a Docker container and accessing it remotely.

We provide the container images as [`moby/buildkit`](https://hub.docker.com/r/moby/buildkit/tags/):

-   `moby/buildkit:latest`: built from the latest regular [release](https://github.com/moby/buildkit/releases)
-   `moby/buildkit:rootless`: same as `latest` but runs as an unprivileged user, see [`docs/rootless.md`](docs/rootless.md)
-   `moby/buildkit:master`: built from the master branch
-   `moby/buildkit:master-rootless`: same as master but runs as an unprivileged user, see [`docs/rootless.md`](docs/rootless.md)

To run daemon in a container:

```bash
docker run -d --name buildkitd --privileged moby/buildkit:latest
export BUILDKIT_HOST=docker-container://buildkitd
buildctl build --help
```

### Kubernetes

For Kubernetes deployments, see [`examples/kubernetes`](./examples/kubernetes).

### Daemonless

To run client and an ephemeral daemon in a single container ("daemonless mode"):

```bash
docker run \
    -it \
    --rm \
    --privileged \
    -v /path/to/dir:/tmp/work \
    --entrypoint buildctl-daemonless.sh \
    moby/buildkit:master \
        build \
        --frontend dockerfile.v0 \
        --local context=/tmp/work \
        --local dockerfile=/tmp/work
```

or

```bash
docker run \
    -it \
    --rm \
    --security-opt seccomp=unconfined \
    --security-opt apparmor=unconfined \
    -e BUILDKITD_FLAGS=--oci-worker-no-process-sandbox \
    -v /path/to/dir:/tmp/work \
    --entrypoint buildctl-daemonless.sh \
    moby/buildkit:master-rootless \
        build \
        --frontend \
        dockerfile.v0 \
        --local context=/tmp/work \
        --local dockerfile=/tmp/work
```

## Opentracing support

BuildKit supports opentracing for buildkitd gRPC API and buildctl commands. To capture the trace to [Jaeger](https://github.com/jaegertracing/jaeger), set `JAEGER_TRACE` environment variable to the collection address.

```bash
docker run -d -p6831:6831/udp -p16686:16686 jaegertracing/all-in-one:latest
export JAEGER_TRACE=0.0.0.0:6831
# restart buildkitd and buildctl so they know JAEGER_TRACE
# any buildctl command should be traced to http://127.0.0.1:16686/
```

## Running BuildKit without root privileges

Please refer to [`docs/rootless.md`](docs/rootless.md).

## Building multi-platform images

See [`docker buildx` documentation](https://github.com/docker/buildx#building-multi-platform-images)

## Contributing

Want to contribute to BuildKit? Awesome! You can find information about contributing to this project in the [CONTRIBUTING.md](/.github/CONTRIBUTING.md)

