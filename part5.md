### ENTRYPOINT

[Dockerfile reference for the ENTRYPOINT instruction](../../engine/reference/builder.md#entrypoint)

The best use for `ENTRYPOINT` is to set the image's main command, allowing that
image to be run as though it was that command (and then use `CMD` as the
default flags).

Let's start with an example of an image for the command line tool `s3cmd`:

```dockerfile
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```

Now the image can be run like this to show the command's help:

```bash
$ docker run s3cmd
```

Or using the right parameters to execute a command:

```bash
$ docker run s3cmd ls s3://mybucket
```

This is useful because the image name can double as a reference to the binary as
shown in the command above.

The `ENTRYPOINT` instruction can also be used in combination with a helper
script, allowing it to function in a similar way to the command above, even
when starting the tool may require more than one step.

For example, the [Postgres Official Image](https://hub.docker.com/_/postgres/)
uses the following script as its `ENTRYPOINT`:

```bash
#!/bin/bash
set -e

if [ "$1" = 'postgres' ]; then
    chown -R postgres "$PGDATA"

    if [ -z "$(ls -A "$PGDATA")" ]; then
        gosu postgres initdb
    fi

    exec gosu postgres "$@"
fi

exec "$@"
```

> Configure app as PID 1
>
> This script uses [the `exec` Bash command](http://wiki.bash-hackers.org/commands/builtin/exec)
> so that the final running application becomes the container's PID 1. This
> allows the application to receive any Unix signals sent to the container.
> For more, see the [`ENTRYPOINT` reference](../../engine/reference/builder.md#entrypoint).

The helper script is copied into the container and run via `ENTRYPOINT` on
container start:

```dockerfile
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["postgres"]
```

This script allows the user to interact with Postgres in several ways.

It can simply start Postgres:

```bash
$ docker run postgres
```

Or, it can be used to run Postgres and pass parameters to the server:

```bash
$ docker run postgres postgres --help
```

Lastly, it could also be used to start a totally different tool, such as Bash:

```bash
$ docker run --rm -it postgres bash
```


### VOLUME

[Dockerfile reference for the VOLUME instruction](../../engine/reference/builder.md#volume)

The `VOLUME` instruction should be used to expose any database storage area,
configuration storage, or files/folders created by your docker container. You
are strongly encouraged to use `VOLUME` for any mutable and/or user-serviceable
parts of your image.

### USER

[Dockerfile reference for the USER instruction](../../engine/reference/builder.md#user)

If a service can run without privileges, use `USER` to change to a non-root
user. Start by creating the user and group in the `Dockerfile` with something
like `RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres`.

> Consider an explicit UID/GID
>
> Users and groups in an image are assigned a non-deterministic UID/GID in that
> the "next" UID/GID is assigned regardless of image rebuilds. So, if it’s
> critical, you should assign an explicit UID/GID.

> Due to an [unresolved bug](https://github.com/golang/go/issues/13548) in the
> Go archive/tar package's handling of sparse files, attempting to create a user
> with a significantly large UID inside a Docker container can lead to disk
> exhaustion because `/var/log/faillog` in the container layer is filled with
> NULL (\0) characters. A workaround is to pass the `--no-log-init` flag to
> useradd. The Debian/Ubuntu `adduser` wrapper does not support this flag.

Avoid installing or using `sudo` as it has unpredictable TTY and
signal-forwarding behavior that can cause problems. If you absolutely need
functionality similar to `sudo`, such as initializing the daemon as `root` but
running it as non-`root`, consider using [“gosu”](https://github.com/tianon/gosu).

Lastly, to reduce layers and complexity, avoid switching `USER` back and forth
frequently.

### WORKDIR

[Dockerfile reference for the WORKDIR instruction](../../engine/reference/builder.md#workdir)

For clarity and reliability, you should always use absolute paths for your
`WORKDIR`. Also, you should use `WORKDIR` instead of  proliferating instructions
like `RUN cd … && do-something`, which are hard to read, troubleshoot, and
maintain.

### ONBUILD

[Dockerfile reference for the ONBUILD instruction](../../engine/reference/builder.md#onbuild)

An `ONBUILD` command executes after the current `Dockerfile` build completes.
`ONBUILD` executes in any child image derived `FROM` the current image.  Think
of the `ONBUILD` command as an instruction the parent `Dockerfile` gives
to the child `Dockerfile`.

A Docker build executes `ONBUILD` commands before any command in a child
`Dockerfile`.

`ONBUILD` is useful for images that are going to be built `FROM` a given
image. For example, you would use `ONBUILD` for a language stack image that
builds arbitrary user software written in that language within the
`Dockerfile`, as you can see in [Ruby’s `ONBUILD` variants](https://github.com/docker-library/ruby/blob/c43fef8a60cea31eb9e7d960a076d633cb62ba8d/2.4/jessie/onbuild/Dockerfile).

Images built with `ONBUILD` should get a separate tag, for example:
`ruby:1.9-onbuild` or `ruby:2.0-onbuild`.

Be careful when putting `ADD` or `COPY` in `ONBUILD`. The "onbuild" image
fails catastrophically if the new build's context is missing the resource being
added. Adding a separate tag, as recommended above, helps mitigate this by
allowing the `Dockerfile` author to make a choice.

## Examples for Official Images

These Official Images have exemplary `Dockerfile`s:

* [Go](https://hub.docker.com/_/golang/)
* [Perl](https://hub.docker.com/_/perl/)
* [Hy](https://hub.docker.com/_/hylang/)
* [Ruby](https://hub.docker.com/_/ruby/)

## Additional resources:

* [Dockerfile Reference](../../engine/reference/builder.md)
* [More about Base Images](baseimages.md)
* [More about Automated Builds](../../docker-hub/builds/index.md)
* [Guidelines for Creating Official Images](../../docker-hub/official_images.md)

