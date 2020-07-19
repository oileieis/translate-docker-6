### CMD

[Dockerfile reference for the CMD instruction](../../engine/reference/builder.md#cmd)

The `CMD` instruction should be used to run the software contained in your
image, along with any arguments. `CMD` should almost always be used in the form
of `CMD ["executable", "param1", "param2"…]`. Thus, if the image is for a
service, such as Apache and Rails, you would run something like `CMD
["apache2","-DFOREGROUND"]`. Indeed, this form of the instruction is recommended
for any service-based image.

In most other cases, `CMD` should be given an interactive shell, such as bash,
python and perl. For example, `CMD ["perl", "-de0"]`, `CMD ["python"]`, or `CMD
["php", "-a"]`. Using this form means that when you execute something like
`docker run -it python`, you’ll get dropped into a usable shell, ready to go.
`CMD` should rarely be used in the manner of `CMD ["param", "param"]` in
conjunction with [`ENTRYPOINT`](../../engine/reference/builder.md#entrypoint), unless
you and your expected users are already quite familiar with how `ENTRYPOINT`
works.

### EXPOSE

[Dockerfile reference for the EXPOSE instruction](../../engine/reference/builder.md#expose)

The `EXPOSE` instruction indicates the ports on which a container listens
for connections. Consequently, you should use the common, traditional port for
your application. For example, an image containing the Apache web server would
use `EXPOSE 80`, while an image containing MongoDB would use `EXPOSE 27017` and
so on.

For external access, your users can execute `docker run` with a flag indicating
how to map the specified port to the port of their choice.
For container linking, Docker provides environment variables for the path from
the recipient container back to the source (ie, `MYSQL_PORT_3306_TCP`).

### ENV

[Dockerfile reference for the ENV instruction](../../engine/reference/builder.md#env)

To make new software easier to run, you can use `ENV` to update the
`PATH` environment variable for the software your container installs. For
example, `ENV PATH /usr/local/nginx/bin:$PATH` ensures that `CMD ["nginx"]`
just works.

The `ENV` instruction is also useful for providing required environment
variables specific to services you wish to containerize, such as Postgres’s
`PGDATA`.

Lastly, `ENV` can also be used to set commonly used version numbers so that
version bumps are easier to maintain, as seen in the following example:

```dockerfile
ENV PG_MAJOR 9.3
ENV PG_VERSION 9.3.4
RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC /usr/src/postgress && …
ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH
```

Similar to having constant variables in a program (as opposed to hard-coding
values), this approach lets you change a single `ENV` instruction to
auto-magically bump the version of the software in your container.

Each `ENV` line creates a new intermediate layer, just like `RUN` commands. This
means that even if you unset the environment variable in a future layer, it
still persists in this layer and its value can't be dumped. You can test this by
creating a Dockerfile like the following, and then building it.

```dockerfile
FROM alpine
ENV ADMIN_USER="mark"
RUN echo $ADMIN_USER > ./mark
RUN unset ADMIN_USER
```

```bash
$ docker run --rm test sh -c 'echo $ADMIN_USER'

mark
```

To prevent this, and really unset the environment variable, use a `RUN` command
with shell commands, to set, use, and unset the variable all in a single layer.
You can separate your commands with `;` or `&&`. If you use the second method,
and one of the commands fails, the `docker build` also fails. This is usually a
good idea. Using `\` as a line continuation character for Linux Dockerfiles
improves readability. You could also put all of the commands into a shell script
and have the `RUN` command just run that shell script.

```dockerfile
FROM alpine
RUN export ADMIN_USER="mark" \
    && echo $ADMIN_USER > ./mark \
    && unset ADMIN_USER
CMD sh
```

```bash
$ docker run --rm test sh -c 'echo $ADMIN_USER'

```


### ADD or COPY

- [Dockerfile reference for the ADD instruction](../../engine/reference/builder.md#add)
- [Dockerfile reference for the COPY instruction](../../engine/reference/builder.md#copy)

Although `ADD` and `COPY` are functionally similar, generally speaking, `COPY`
is preferred. That’s because it’s more transparent than `ADD`. `COPY` only
supports the basic copying of local files into the container, while `ADD` has
some features (like local-only tar extraction and remote URL support) that are
not immediately obvious. Consequently, the best use for `ADD` is local tar file
auto-extraction into the image, as in `ADD rootfs.tar.xz /`.

If you have multiple `Dockerfile` steps that use different files from your
context, `COPY` them individually, rather than all at once. This ensures that
each step's build cache is only invalidated (forcing the step to be re-run) if
the specifically required files change.

For example:

```dockerfile
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```

Results in fewer cache invalidations for the `RUN` step, than if you put the
`COPY . /tmp/` before it.

Because image size matters, using `ADD` to fetch packages from remote URLs is
strongly discouraged; you should use `curl` or `wget` instead. That way you can
delete the files you no longer need after they've been extracted and you don't
have to add another layer in your image. For example, you should avoid doing
things like:

```dockerfile
ADD http://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```

And instead, do something like:

```dockerfile
RUN mkdir -p /usr/src/things \
    && curl -SL http://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
```

For other items (files, directories) that do not require `ADD`’s tar
auto-extraction capability, you should always use `COPY`.