This document covers recommended best practices and methods for building
efficient images.

Docker builds images automatically by reading the instructions from a
`Dockerfile` -- a text file that contains all commands, in order, needed to
build a given image. A `Dockerfile` adheres to a specific format and set of
instructions which you can find at [Dockerfile reference](../../engine/reference/builder.md).

A Docker image consists of read-only layers each of which represents a
Dockerfile  instruction. The layers are stacked and each one is a delta of the
changes from the previous layer. Consider this `Dockerfile`:

```dockerfile
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

Each instruction creates one layer:

- `FROM` creates a layer from the `ubuntu:18.04` Docker image.
- `COPY` adds files from your Docker client's current directory.
- `RUN` builds your application with `make`.
- `CMD` specifies what command to run within the container.

When you run an image and generate a container, you add a new _writable layer_
(the "container layer") on top of the underlying layers. All changes made to
the running container, such as writing new files, modifying existing files, and
deleting files, are written to this thin writable container layer.

For more on image layers (and how Docker builds and stores images), see
[About storage drivers](../../storage/storagedriver/index.md).

## General guidelines and recommendations

### Create ephemeral containers

The image defined by your `Dockerfile` should generate containers that are as
ephemeral as possible. By "ephemeral", we mean that the container can be stopped
and destroyed, then rebuilt and replaced with an absolute minimum set up and
configuration.

Refer to [Processes](https://12factor.net/processes) under _The Twelve-factor App_
methodology to get a feel for the motivations of running containers in such a
stateless fashion.

### Understand build context

When you issue a `docker build` command, the current working directory is called
the _build context_. By default, the Dockerfile is assumed to be located here,
but you can specify a different location with the file flag (`-f`). Regardless
of where the `Dockerfile` actually lives, all recursive contents of files and
directories in the current directory are sent to the Docker daemon as the build
context.

> Build context example
>
> Create a directory for the build context and `cd` into it. Write "hello" into
> a text file named `hello` and create a Dockerfile that runs `cat` on it. Build
> the image from within the build context (`.`):
>
> ```shell
> mkdir myproject && cd myproject
> echo "hello" > hello
> echo -e "FROM busybox\nCOPY /hello /\nRUN cat /hello" > Dockerfile
> docker build -t helloapp:v1 .
> ```
>
> Move `Dockerfile` and `hello` into separate directories and build a second
> version of the image (without relying on cache from the last build). Use `-f`
> to point to the Dockerfile and specify the directory of the build context:
>
> ```shell
> mkdir -p dockerfiles context
> mv Dockerfile dockerfiles && mv hello context
> docker build --no-cache -t helloapp:v2 -f dockerfiles/Dockerfile context
> ```

Inadvertently including files that are not necessary for building an image
results in a larger build context and larger image size. This can increase the
time to build the image, time to pull and push it, and the container runtime
size. To see how big your build context is, look for a message like this when
building your `Dockerfile`:

```none
Sending build context to Docker daemon  187.8MB
```

### Pipe Dockerfile through `stdin`

Docker has the ability to build images by piping `Dockerfile` through `stdin`
with a _local or remote build context_. Piping a `Dockerfile` through `stdin`
can be useful to perform one-off builds without writing a Dockerfile to disk,
or in situations where the `Dockerfile` is generated, and should not persist
afterwards.

> The examples in this section use [here documents](http://tldp.org/LDP/abs/html/here-docs.html)
> for convenience, but any method to provide the `Dockerfile` on `stdin` can be
> used.
> 
> For example, the following commands are equivalent: 
> 
> ```bash
> echo -e 'FROM busybox\nRUN echo "hello world"' | docker build -
> ```
> 
> ```bash
> docker build -<<EOF
> FROM busybox
> RUN echo "hello world"
> EOF
> ```
> 
> You can substitute the examples with your preferred approach, or the approach
> that best fits your use-case.


#### Build an image using a Dockerfile from stdin, without sending build context

Use this syntax to build an image using a `Dockerfile` from `stdin`, without
sending additional files as build context. The hyphen (`-`) takes the position
of the `PATH`, and instructs Docker to read the build context (which only
contains a `Dockerfile`) from `stdin` instead of a directory:

```bash
docker build [OPTIONS] -
```

The following example builds an image using a `Dockerfile` that is passed through
`stdin`. No files are sent as build context to the daemon.

```bash
docker build -t myimage:latest -<<EOF
FROM busybox
RUN echo "hello world"
EOF
```

Omitting the build context can be useful in situations where your `Dockerfile`
does not require files to be copied into the image, and improves the build-speed,
as no files are sent to the daemon.

If you want to improve the build-speed by excluding _some_ files from the build-
context, refer to [exclude with .dockerignore](#exclude-with-dockerignore).

> **Note**: Attempting to build a Dockerfile that uses `COPY` or `ADD` will fail
> if this syntax is used. The following example illustrates this:
> 
> ```bash
> # create a directory to work in
> mkdir example
> cd example
> 
> # create an example file
> touch somefile.txt
> 
> docker build -t myimage:latest -<<EOF
> FROM busybox
> COPY somefile.txt .
> RUN cat /somefile.txt
> EOF
> 
> # observe that the build fails
> ...
> Step 2/3 : COPY somefile.txt .
> COPY failed: stat /var/lib/docker/tmp/docker-builder249218248/somefile.txt: no such file or directory
> ```

#### Build from a local build context, using a Dockerfile from stdin

Use this syntax to build an image using files on your local filesystem, but using
a `Dockerfile` from `stdin`. The syntax uses the `-f` (or `--file`) option to
specify the `Dockerfile` to use, using a hyphen (`-`) as filename to instruct
Docker to read the `Dockerfile` from `stdin`:

```bash
docker build [OPTIONS] -f- PATH
```

The example below uses the current directory (`.`) as the build context, and builds
an image using a `Dockerfile` that is passed through `stdin` using a [here
document](http://tldp.org/LDP/abs/html/here-docs.html).

```bash
# create a directory to work in
mkdir example
cd example

# create an example file
touch somefile.txt

# build an image using the current directory as context, and a Dockerfile passed through stdin
docker build -t myimage:latest -f- . <<EOF
FROM busybox
COPY somefile.txt .
RUN cat /somefile.txt
EOF
```

#### Build from a remote build context, using a Dockerfile from stdin

Use this syntax to build an image using files from a remote `git` repository, 
using a `Dockerfile` from `stdin`. The syntax uses the `-f` (or `--file`) option to
specify the `Dockerfile` to use, using a hyphen (`-`) as filename to instruct
Docker to read the `Dockerfile` from `stdin`:

```bash
docker build [OPTIONS] -f- PATH
```

This syntax can be useful in situations where you want to build an image from a
repository that does not contain a `Dockerfile`, or if you want to build with a custom
`Dockerfile`, without maintaining your own fork of the repository.

The example below builds an image using a `Dockerfile` from `stdin`, and adds
the `hello.c` file from the ["hello-world" Git repository on GitHub](https://github.com/docker-library/hello-world).

```bash
docker build -t myimage:latest -f- https://github.com/docker-library/hello-world.git <<EOF
FROM busybox
COPY hello.c .
EOF
```

> **Under the hood**
>
> When building an image using a remote Git repository as build context, Docker 
> performs a `git clone` of the repository on the local machine, and sends
> those files as build context to the daemon. This feature requires `git` to be
> installed on the host where you run the `docker build` command.