---
title: "Alpine Linux Containers"
description: >-
  Using Alpine Linux Docker containers as a lightweight alternative
  to traditional Debian-based containers.
tags:
  - docker
---

## What is Alpine Linux?

Many official Docker container images come in an `alpine` variant.
They are based on [Alpine Linux][alpine], an incredibly lightweight distribution
that results in smaller image sizes. For example, the official Alpine-based
Golang image is almost 3x smaller than its Debian-based counterpart:

```bash
docker pull golang:1.19-bullseye
docker pull golang:1.19-alpine
docker images --filter 'reference=golang' \
    --format '{{ .Repository }}:{{ .Tag }}\t{{ .Size }}'
```

```
golang:1.19-bullseye    993MB
golang:1.19-alpine      354MB
```

Alpine Linux achieves these gains by omitting many packages that are useful
for a desktop OS, but are less important for a container image. The distribution
uses musl libc instead of glibc, and busybox instead of dash or bash.
Unfortunately, the barebones environment does result in a steeper learning curve
and may cause incompatibilities with some projects.

<!-- truncate -->

## Working with Alpine Images

### Package Management

Alpine [packages] are installed via `apk`, a process that is arguably simpler than
Debian's `apt-get`:

```docker
FROM golang:1.19-alpine

RUN apk add --no-cache gcc musl-dev
```

The `--no-cache` flag prevents temporary files from being written to the container image
and saves us from needing additional commands to sync the local package cache and clean it
when done.

Sometimes a set of packages is temporarily required as a build dependency, not a runtime dependency.
To uninstall packages, use `apk del`:

```docker
FROM golang:1.19-alpine

RUN apk add --no-cache gcc musl-dev \
    && go build ./... \
    && apk del gcc musl-dev
```

Repeating the package list in both the `add` and `del` commands is cumbersome. Instead, we can rewrite
this operation to use a _virtual package_:

```docker
FROM golang:1.19-alpine

RUN apk add --no-cache --virtual .build-deps gcc musl-dev \
    && go build ./... \
    && apk del .build-deps
```

Here, we create a new package called `.build-deps`, and install `gcc` and `musl-dev` under it.
Then, deletion can reference `.build-deps` instead of the individual names.

### Entrypoint

Docker containers should have an init process as the entrypoint. The init process is PID 1 --
the first process in the tree. It must be able to forward signals and clean up zombie processes,
responsibilities that are usually not left to ordinary applications.

On Alpine, a good init system to use is `tini`:

```docker
FROM golang:1.19-alpine

RUN apk add --no-cache tini

ENTRYPOINT ["/sbin/tini", "--"]
```

### Alpine Releases

At any given moment, there are several actively-supported [release branches] of Alpine Linux.
It is usually best to use the latest version. However, if you do need to pin a specific release,
most official Docker repositories offer those image tags. For example, in the above dockerfiles,
we could have used `golang:1.19-alpine3.17` in place of `golang:1.19-alpine`.

## Musl Libc

The largest obstacle to widespread deployment of Alpine Linux is musl libc. Musl is an alternate
implementation of the C standard library, which is [not perfectly compatible][musl] with glibc.
As a result, some programs designed to work with glibc may fail to compile or may crash under musl.

Musl compatibility may not be a concern in simple projects. However, even if your own code
doesn't interact directly with the C standard library, your subdependencies and frameworks might.
For instance, Python packages are distributed as "wheels" that use the `manylinux` standard
([PEP 600][pep600]), designed specifically for glibc. `musllinux` wheels exist, but are less widely adopted.
Whenever a compatible wheel cannot be downloaded, Python needs to install a package locally
from source. This process is slower and requires more compile-time dependencies, especially for
components like database drivers.

Similar issues exist in the Node.js ecosystem. Browser drivers and frontend test runners are
difficult to deploy in a musl environment due to the use of prebuilt binaries.

## The Benefits of Alpine

If you can build and run your project under Alpine Linux, it is worth doing so just to speed up your
CI/CD pipeline. Smaller images take less time to upload to a container registry, and less time to
download onto your production hosts. The benefits add up quickly as your environment scales up to
handle multiple containers at a time.

Aside from efficiency, there is also a security benefit. Since Alpine containers have so few packages
preinstalled, the attack surface is much smaller compared to other container distributions.
So, the next time you need to dockerize your project, consider moving beyond a default Debian image.


[alpine]: https://alpinelinux.org/
[musl]: https://wiki.musl-libc.org/functional-differences-from-glibc.html
[packages]: https://pkgs.alpinelinux.org/packages
[pep600]: https://peps.python.org/pep-0600/
[release branches]: https://alpinelinux.org/releases/
