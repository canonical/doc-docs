# DevOps Centre (DOC) documentation repository

How to contribute.

* [juju.is/tutorials mapper](https://discourse.charmhub.io/t/about-the-tutorials-category/2628):
This topic tracks the published tutorials in http://juju.is/tutorials. A template can be found
at `templates/juju-tutorials.md`. Copy it to `juju-tutorials/<title>.md`, edit the content and
create a PR based on your change. An automatic lint checker will be run when a PR is created.

* [ubuntu.com/tutorials mapper](https://discourse.ubuntu.com/t/about-the-tutorials-category/13611):
This topic tracks the published tutorials in https://ubuntu.com/tutorials. A template can be found
at `templates/ubuntu-tutorials.md`. Copy it to `ubuntu-tutorials/<title>.md`, edit the content and
create a PR based on your change. An automatic lint checker will be run when a PR is created.

## Markdown lint checker for Juju tutorials

Local testing can be run from the CLI via:

```bash
docker run --rm -v "$(pwd)/:/data:ro" avtodev/markdown-lint:v1 \
  --config /data/lint/config/tutorials.yml \
  /data/juju-tutorials/**.md /data/samples/juju-tutorials-nextcloud-collabora.md /data/templates/juju-tutorials.md
```

## Markdown lint checker for Ubuntu tutorials

Local testing can be run from the CLI via:

```bash
docker run --rm -v "$(pwd)/:/data:ro" avtodev/markdown-lint:v1 \
  --config /data/lint/config/tutorials.yml \
  /data/ubuntu-tutorials/**.md /data/samples/ubuntu-tutorials-howto-write-a-tutorial.md /data/templates/ubuntu-tutorials.md
```

# References

More information about the markdown lint checker can be found at
[Docker Hub](https://hub.docker.com/r/avtodev/markdown-lint) and
[GitHub](https://github.com/avto-dev/markdown-lint).
