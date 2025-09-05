---
title: "An Intro to Dev Containers in VS Code"
description: >-
  Using Dev Containers in Visual Studio Code to provide reproducible
  local development environments.
tags:
  - docker
---

## The Problem with Host Workflows

Running a development instance of your application often means installing its dependencies on
your host computer. You'll need to consider operating system packages, compilers, and language-specific frameworks.
For example, even a simple project like this blog site requires a particular version of Ruby, Jekyll, and other gems.

Problems arise when you have larger projects with complex installation requirements, or multiple projects with
conflicting dependencies. Framework-level issues can be solved with tools like RVM, which allow the developer to
switch between multiple concurrent installations of Ruby. OS-level issues are often trickier. For instance,
on Debian Bullseye, `apt-get install gcc` provides GCC version 10. What if your project requires a newer or older compiler?

On the deployment side we have solved these problems using Docker containers. Each application can have its own
set of container images with an isolated environment. We can apply the same techniques for local development.

<!-- truncate -->

## Dev Container Basics

The dev container feature in Visual Studio Code allows you to open a project inside a Docker container, while still having
access to all IDE features like extensions and debuggers. You can run commands in the container through the terminal tab.
VS Code automatically manages network port mappings, so servers will still be accessible on `localhost`.

Your dev container image can be one of the provided reference images, or it can be based off your project's existing container.
The latter is typically more useful, since it allows you to keep your dev image consistent with your existing production one.

To use dev containers, create the `.devcontainer/devcontainer.json` file in your project directory.
Here is a minimal example. (VS Code uses "JSON with Comments", so these snippets are valid.)

```json-doc
{
    "name": "Existing Dockerfile",
    "build": {
        // The build context, relative to the location of this config file.
        // (so ".." means the project root directory)
        "context": "..",

        // The path to the project's existing Dockerfile,
        // relative to the location of this config file.
        "dockerfile": "../Dockerfile"
    },
    "features": {},

    // VS Code extensions to install in the container after building it.
    // For example, a Python project might include the following:
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-python.python",
                "ms-python.vscode-pylance"
            ]
        }
    }
```

Once you have a valid `devcontainer.json`, VS Code will prompt you to reopen the project in the container.
Or, you can access the full set of dev container options from the command pallete.

## Using Docker Compose

For more mature projects, you may want to use Docker Compose. This approach enables you to specify dependencies
on additional service containers (like databases) and inject overrides. Here is a standard file layout:

```
.
├── .devcontainer
│   ├── devcontainer.json
│   └── docker-compose.yml  // overrides for the dev container
├── Dockerfile
├── README.md
└── docker-compose.yml  // the project's existing compose file
```

An example of my Jekyll project's compose file:
```yaml
services:
  jekyll:
    image: csang/jekyll:latest
    build:
      context: .
      dockerfile: ./Dockerfile
```

The dev container config:
```json-doc
{
    "name": "Existing Docker Compose (Extend)",

    "dockerComposeFile": [
        "../docker-compose.yml",  // the project's compose file
        "docker-compose.yml"  // your override file
    ],
    "service": "jekyll",
    "runServices": ["jekyll"],
    "workspaceFolder": "/workspaces/${localWorkspaceFolderBasename}"
}
```

Use the override file to control the dev container's volume mounts.
You should always have a bind mount corresponding to the project directory (`.`),
so that edits to the source code in the container are reflected on the host filesystem.
The target directory must match the `workspaceFolder` in `devcontainer.json`.

Add a named volume for the VS Code extensions directory to prevent extension reinstalls
whenever the dev container is restarted.

```yaml
services:
  jekyll:
    volumes:
      - .:/workspaces/csdev.github.io:cached
      - jekyll-vscode-ext:/root/.vscode-server/extensions

    # Overrides default command so things don't shut down after the process ends.
    # (This is provided by VS Code - don't modify it.)
    command: /bin/sh -c "while sleep 1000; do :; done"

volumes:
  jekyll-vscode-ext:
```

## File Ownership

Most container images only ship with a root user. If you develop with this user, it will cause
bind-mounted files on the host system to become owned by root, requiring you to hack together
`chown` commands to fix the broken permissions.

To avoid this problem, create a non-privileged user in the container with a UID and GID matching
the ones on your host. For example, in your Dockerfile:

```docker
FROM ruby:2.7.4-slim-bullseye
ARG UID=1000 GID=1000

RUN groupadd --gid "$GID" jekyll \
    && useradd --uid "$UID" --gid "$GID" -m jekyll

USER jekyll
COPY --chown=jekyll site/Gemfile site/Gemfile.lock ./
```

Run `id -u` and `id -g` to get your actual host UID and GID (usually 1000 on Ubuntu).
The `USER` instruction switches the default user of your container and should come after any operations
that require root privileges (such as `apt-get install`). When copying files into the container,
use `COPY --chown` to set the correct owner.

## Final Thoughts

With a little extra preparation, you can create Docker container images usable for both production deployments
and local development. Standardizing your environments will hopefully reduce all those _works on my machine_ moments.
Finally, since dev containers are fully configurable through code, it is easy to experiment, sync your settings,
and share your setup with others.
