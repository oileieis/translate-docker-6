## Dockerfile instructions

These recommendations are designed to help you create an efficient and
maintainable `Dockerfile`.

### FROM

[Dockerfile reference for the FROM instruction](../../engine/reference/builder.md#from)

Whenever possible, use current official images as the basis for your
images. We recommend the [Alpine image](https://hub.docker.com/_/alpine/) as it
is tightly controlled and small in size (currently under 5 MB), while still
being a full Linux distribution.

### LABEL

[Understanding object labels](../../config/labels-custom-metadata.md)

You can add labels to your image to help organize images by project, record
licensing information, to aid in automation, or for other reasons. For each
label, add a line beginning with `LABEL` and with one or more key-value pairs.
The following examples show the different acceptable formats. Explanatory comments are included inline.

> Strings with spaces must be quoted **or** the spaces must be escaped. Inner
> quote characters (`"`), must also be escaped.

```dockerfile
# Set one or more individual labels
LABEL com.example.version="0.0.1-beta"
LABEL vendor1="ACME Incorporated"
LABEL vendor2=ZENITH\ Incorporated
LABEL com.example.release-date="2015-02-12"
LABEL com.example.version.is-production=""
```

An image can have more than one label. Prior to Docker 1.10, it was recommended
to combine all labels into a single `LABEL` instruction, to prevent extra layers
from being created. This is no longer necessary, but combining labels is still
supported.

```dockerfile
# Set multiple labels on one line
LABEL com.example.version="0.0.1-beta" com.example.release-date="2015-02-12"
```

The above can also be written as:

```dockerfile
# Set multiple labels at once, using line-continuation characters to break long lines
LABEL vendor=ACME\ Incorporated \
      com.example.is-beta= \
      com.example.is-production="" \
      com.example.version="0.0.1-beta" \
      com.example.release-date="2015-02-12"
```

See [Understanding object labels](../../config/labels-custom-metadata.md)
for guidelines about acceptable label keys and values. For information about
querying labels, refer to the items related to filtering in
[Managing labels on objects](../../config/labels-custom-metadata.md#manage-labels-on-objects).
See also [LABEL](../../engine/reference/builder.md#label) in the Dockerfile reference.

### RUN

[Dockerfile reference for the RUN instruction](../../engine/reference/builder.md#run)

Split long or complex `RUN` statements on multiple lines separated with
backslashes to make your `Dockerfile` more readable, understandable, and
maintainable.

#### apt-get

Probably the most common use-case for `RUN` is an application of `apt-get`.
Because it installs packages, the `RUN apt-get` command has several gotchas to
look out for.

Avoid `RUN apt-get upgrade` and `dist-upgrade`, as many of the "essential"
packages from the parent images cannot upgrade inside an
[unprivileged container](../../engine/reference/run.md#security-configuration). If a package
contained in the parent image is out-of-date, contact its maintainers. If you
know there is a particular package, `foo`, that needs to be updated, use
`apt-get install -y foo` to update automatically.

Always combine `RUN apt-get update` with `apt-get install` in the same `RUN`
statement. For example:

```dockerfile
RUN apt-get update && apt-get install -y \
    package-bar \
    package-baz \
    package-foo
```

Using `apt-get update` alone in a `RUN` statement causes caching issues and
subsequent `apt-get install` instructions fail. For example, say you have a
Dockerfile:

```dockerfile
FROM ubuntu:18.04
RUN apt-get update
RUN apt-get install -y curl
```

After building the image, all layers are in the Docker cache. Suppose you later
modify `apt-get install` by adding extra package:

```dockerfile
FROM ubuntu:18.04
RUN apt-get update
RUN apt-get install -y curl nginx
```

Docker sees the initial and modified instructions as identical and reuses the
cache from previous steps. As a result the `apt-get update` is _not_ executed
because the build uses the cached version. Because the `apt-get update` is not
run, your build can potentially get an outdated version of the `curl` and
`nginx` packages.

Using `RUN apt-get update && apt-get install -y` ensures your Dockerfile
installs the latest package versions with no further coding or manual
intervention. This technique is known as "cache busting". You can also achieve
cache-busting by specifying a package version. This is known as version pinning,
for example:

```dockerfile
RUN apt-get update && apt-get install -y \
    package-bar \
    package-baz \
    package-foo=1.3.*
```

Version pinning forces the build to retrieve a particular version regardless of
what’s in the cache. This technique can also reduce failures due to unanticipated changes
in required packages.

Below is a well-formed `RUN` instruction that demonstrates all the `apt-get`
recommendations.

```dockerfile
RUN apt-get update && apt-get install -y \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
 && rm -rf /var/lib/apt/lists/*
```

The `s3cmd` argument specifies a version `1.1.*`. If the image previously
used an older version, specifying the new one causes a cache bust of `apt-get
update` and ensures the installation of the new version. Listing packages on
each line can also prevent mistakes in package duplication.

In addition, when you clean up the apt cache by removing `/var/lib/apt/lists` it
reduces the image size, since the apt cache is not stored in a layer. Since the
`RUN` statement starts with `apt-get update`, the package cache is always
refreshed prior to `apt-get install`.

> Official Debian and Ubuntu images [automatically run `apt-get clean`](https://github.com/moby/moby/blob/03e2923e42446dbb830c654d0eec323a0b4ef02a/contrib/mkimage/debootstrap#L82-L105),
> so explicit invocation is not required.

#### Using pipes

Some `RUN` commands depend on the ability to pipe the output of one command into another, using the pipe character (`|`), as in the following example:

```dockerfile
RUN wget -O - https://some.site | wc -l > /number
```

Docker executes these commands using the `/bin/sh -c` interpreter, which only
evaluates the exit code of the last operation in the pipe to determine success.
In the example above this build step succeeds and produces a new image so long
as the `wc -l` command succeeds, even if the `wget` command fails.

If you want the command to fail due to an error at any stage in the pipe,
prepend `set -o pipefail &&` to ensure that an unexpected error prevents the
build from inadvertently succeeding. For example:

```dockerfile
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```
> Not all shells support the `-o pipefail` option.
>
> In cases such as the `dash` shell on
> Debian-based images, consider using the _exec_ form of `RUN` to explicitly
> choose a shell that does support the `pipefail` option. For example:
>
> ```dockerfile
> RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]
> ```