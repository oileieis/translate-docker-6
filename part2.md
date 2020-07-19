### Exclude with .dockerignore

To exclude files not relevant to the build (without restructuring your source
repository) use a `.dockerignore` file. This file supports exclusion patterns
similar to `.gitignore` files. For information on creating one, see the
[.dockerignore file](../../engine/reference/builder.md#dockerignore-file).

### Use multi-stage builds

[Multi-stage builds](multistage-build.md) allow you to drastically reduce the
size of your final image, without struggling to reduce the number of intermediate
layers and files.

Because an image is built during the final stage of the build process, you can
minimize image layers by [leveraging build cache](#leverage-build-cache).

For example, if your build contains several layers, you can order them from the
less frequently changed (to ensure the build cache is reusable) to the more
frequently changed:

* Install tools you need to build your application

* Install or update library dependencies

* Generate your application

A Dockerfile for a Go application could look like:

```dockerfile
FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
```

### Don't install unnecessary packages

To reduce complexity, dependencies, file sizes, and build times, avoid
installing extra or unnecessary packages just because they might be "nice to
have." For example, you don’t need to include a text editor in a database image.

### Decouple applications

Each container should have only one concern. Decoupling applications into
multiple containers makes it easier to scale horizontally and reuse containers.
For instance, a web application stack might consist of three separate
containers, each with its own unique image, to manage the web application,
database, and an in-memory cache in a decoupled manner.

Limiting each container to one process is a good rule of thumb, but it is not a
hard and fast rule. For example, not only can containers be
[spawned with an init process](../../engine/reference/run.md#specify-an-init-process),
some programs might spawn additional processes of their own accord. For
instance, [Celery](http://www.celeryproject.org/) can spawn multiple worker
processes, and [Apache](https://httpd.apache.org/) can create one process per
request.

Use your best judgment to keep containers as clean and modular as possible. If
containers depend on each other, you can use [Docker container networks](../../network/index.md)
to ensure that these containers can communicate.

### Minimize the number of layers

In older versions of Docker, it was important that you minimized the number of
layers in your images to ensure they were performant. The following features
were added to reduce this limitation:

- Only the instructions `RUN`, `COPY`, `ADD` create layers. Other instructions
  create temporary intermediate images, and do not increase the size of the build.

- Where possible, use [multi-stage builds](multistage-build.md), and only copy
  the artifacts you need into the final image. This allows you to include tools
  and debug information in your intermediate build stages without increasing the
  size of the final image.

### Sort multi-line arguments

Whenever possible, ease later changes by sorting multi-line arguments
alphanumerically. This helps to avoid duplication of packages and make the
list much easier to update. This also makes PRs a lot easier to read and
review. Adding a space before a backslash (`\`) helps as well.

Here’s an example from the [`buildpack-deps` image](https://github.com/docker-library/buildpack-deps):

```dockerfile
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
```

### Leverage build cache

When building an image, Docker steps through the instructions in your
`Dockerfile`, executing each in the order specified. As each instruction is
examined, Docker looks for an existing image in its cache that it can reuse,
rather than creating a new (duplicate) image.

If you do not want to use the cache at all, you can use the `--no-cache=true`
option on the `docker build` command. However, if you do let Docker use its
cache, it is important to understand when it can, and cannot, find a matching
image. The basic rules that Docker follows are outlined below:

- Starting with a parent image that is already in the cache, the next
  instruction is compared against all child images derived from that base
  image to see if one of them was built using the exact same instruction. If
  not, the cache is invalidated.

- In most cases, simply comparing the instruction in the `Dockerfile` with one
  of the child images is sufficient. However, certain instructions require more
  examination and explanation.

- For the `ADD` and `COPY` instructions, the contents of the file(s)
  in the image are examined and a checksum is calculated for each file.
  The last-modified and last-accessed times of the file(s) are not considered in
  these checksums. During the cache lookup, the checksum is compared against the
  checksum in the existing images. If anything has changed in the file(s), such
  as the contents and metadata, then the cache is invalidated.

- Aside from the `ADD` and `COPY` commands, cache checking does not look at the
  files in the container to determine a cache match. For example, when processing
  a `RUN apt-get -y update` command the files updated in the container
  are not examined to determine if a cache hit exists.  In that case just
  the command string itself is used to find a match.

Once the cache is invalidated, all subsequent `Dockerfile` commands generate new
images and the cache is not used.