# Docker, Dockerfiles, and Linux Commands — A Practical Guide

This guide explains how Dockerfiles, Linux commands, package managers, image layers, networking, caching, and container startup behavior work together.

It is written for developers who already understand application development but want a stronger mental model of what actually happens when Docker builds and runs a container.

---

## Table of contents

1. [The core Docker mental model](#1-the-core-docker-mental-model)
2. [Image vs container](#2-image-vs-container)
3. [What a Dockerfile does](#3-what-a-dockerfile-does)
4. [Dockerfile instructions vs Linux commands](#4-dockerfile-instructions-vs-linux-commands)
5. [Understanding `FROM`](#5-understanding-from)
6. [Understanding Ubuntu and APT](#6-understanding-ubuntu-and-apt)
7. [`apt-get update`, `install`, and `upgrade`](#7-apt-get-update-install-and-upgrade)
8. [Why APT commands are combined](#8-why-apt-commands-are-combined)
9. [Shell operators used in Dockerfiles](#9-shell-operators-used-in-dockerfiles)
10. [Docker image layers](#10-docker-image-layers)
11. [Docker build caching](#11-docker-build-caching)
12. [Why cleanup must happen in the same layer](#12-why-cleanup-must-happen-in-the-same-layer)
13. [Should you run `apt-get upgrade`?](#13-should-you-run-apt-get-upgrade)
14. [Understanding `WORKDIR`, `COPY`, `ARG`, and `ENV`](#14-understanding-workdir-copy-arg-and-env)
15. [Understanding `RUN`, `CMD`, and `ENTRYPOINT`](#15-understanding-run-cmd-and-entrypoint)
16. [Understanding `EXPOSE`](#16-understanding-expose)
17. [Port publishing and networking](#17-port-publishing-and-networking)
18. [Who uses `EXPOSE` metadata?](#18-who-uses-expose-metadata)
19. [Multi-stage builds](#19-multi-stage-builds)
20. [A production-style .NET Dockerfile](#20-a-production-style-net-dockerfile)
21. [Common Linux commands in Dockerfiles](#21-common-linux-commands-in-dockerfiles)
22. [Users, ownership, and permissions](#22-users-ownership-and-permissions)
23. [Docker build context and `.dockerignore`](#23-docker-build-context-and-dockerignore)
24. [Container filesystem behavior](#24-container-filesystem-behavior)
25. [Volumes and bind mounts](#25-volumes-and-bind-mounts)
26. [Environment variables and secrets](#26-environment-variables-and-secrets)
27. [Signals, PID 1, and graceful shutdown](#27-signals-pid-1-and-graceful-shutdown)
28. [Container health checks](#28-container-health-checks)
29. [Image tags, digests, and reproducibility](#29-image-tags-digests-and-reproducibility)
30. [Architectures: `amd64` and `arm64`](#30-architectures-amd64-and-arm64)
31. [Security and vulnerability scanners](#31-security-and-vulnerability-scanners)
32. [Chiseled and distroless images](#32-chiseled-and-distroless-images)
33. [Debugging useful commands](#33-debugging-useful-commands)
34. [Common Docker mistakes](#34-common-docker-mistakes)
35. [Final mental model](#35-final-mental-model)
36. [Official references](#36-official-references)

---

# 1. The core Docker mental model

Docker is easier to understand when you separate four concepts:

```text
Dockerfile
    ↓ docker build
Image
    ↓ docker run
Container
    ↓ starts
Application process
```

A **Dockerfile** is a recipe.

An **image** is the packaged result of executing that recipe.

A **container** is a running or stopped instance created from the image.

The **application process** is the command running inside the container.

For example:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY . .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

Then:

```bash
docker build -t myapp .
docker run myapp
```

Conceptually:

```text
Read Dockerfile
    ↓
Create an immutable image
    ↓
Create a container from the image
    ↓
Start: dotnet MyApp.dll
```

---

# 2. Image vs container

An image and a container are not the same thing.

## Image

An image is:

- immutable
- reusable
- composed of layers
- stored locally or in a registry
- used as a template for containers

Examples:

```text
ubuntu:24.04
mcr.microsoft.com/dotnet/aspnet:10.0
my-api:1.4.2
```

## Container

A container is:

- an instance of an image
- given a small writable filesystem layer
- assigned networking and process isolation
- started with a specific command
- disposable by design

You can create many containers from one image:

```text
my-api:1.0 image
    ├── container A
    ├── container B
    └── container C
```

This is what ECS does when an ECS service runs multiple tasks from the same container image.

---

# 3. What a Dockerfile does

A Dockerfile is not simply a Bash script.

It contains **Dockerfile instructions**. Some instructions execute Linux commands, while others configure image metadata.

Example:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && apt-get install -y curl

WORKDIR /app

COPY . .

ENV APP_ENVIRONMENT=production

EXPOSE 8080

CMD ["./myapp"]
```

Docker reads it from top to bottom.

Each instruction changes either:

1. the image filesystem, or
2. the image configuration metadata.

---

# 4. Dockerfile instructions vs Linux commands

This distinction is fundamental.

## Dockerfile instructions

These are interpreted by the Docker builder:

```dockerfile
FROM
RUN
COPY
ADD
WORKDIR
ENV
ARG
USER
EXPOSE
CMD
ENTRYPOINT
HEALTHCHECK
```

## Linux commands

These run inside a build environment when used through `RUN`:

```dockerfile
RUN apt-get update
RUN mkdir -p /app/data
RUN chmod +x /app/start.sh
RUN rm -rf /tmp/files
```

In this example:

```dockerfile
RUN apt-get update
```

- `RUN` is a Dockerfile instruction.
- `apt-get update` is a Linux command executed inside the temporary build container.

---

# 5. Understanding `FROM`

```dockerfile
FROM ubuntu:24.04
```

This means:

> Start this image using the filesystem and configuration contained in the `ubuntu:24.04` image.

Docker does not install Ubuntu like a virtual machine installation.

The Ubuntu image already contains a small Linux userspace filesystem:

```text
/bin
/etc
/lib
/usr
/var
```

It does not contain a separate Linux kernel.

Containers use the host's Linux kernel. The image provides the userspace libraries, commands, configuration, and application files.

For .NET:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0
```

The image normally contains:

- a Linux userspace
- the .NET runtime
- ASP.NET Core runtime libraries
- OS libraries needed by .NET
- certificate configuration
- one or more predefined users

It does not normally contain the full .NET SDK.

---

# 6. Understanding Ubuntu and APT

Ubuntu and Debian use the APT package-management system.

APT manages:

- package repository configuration
- local package catalogues
- downloaded `.deb` packages
- dependencies
- installed package records

Repository configuration is commonly found under:

```text
/etc/apt/sources.list
/etc/apt/sources.list.d/
```

Downloaded package indexes are commonly stored under:

```text
/var/lib/apt/lists/
```

Installed package metadata is maintained under:

```text
/var/lib/dpkg/
```

APT is a higher-level package manager. Underneath, `dpkg` performs lower-level Debian package installation and database operations.

---

# 7. `apt-get update`, `install`, and `upgrade`

## `apt-get update`

```bash
apt-get update
```

This refreshes the local package catalogue.

It downloads current metadata from configured package repositories, including:

- package names
- available versions
- dependency information
- download locations
- package checksums

It does **not** update installed libraries.

Mental model:

```text
apt-get update
=
Refresh the available-products catalogue
```

## `apt-get install`

```bash
apt-get install -y curl
```

This:

1. searches the local package catalogue
2. resolves the requested version
3. resolves required dependencies
4. downloads package archives
5. installs and configures them

Installing one package may install several dependency packages.

For example:

```bash
apt-get install -y curl
```

could also bring packages such as:

```text
libcurl
libnghttp2
certificate-related dependencies
compression libraries
```

The exact dependency list depends on the Linux distribution and package version.

## `apt-get upgrade`

```bash
apt-get upgrade -y
```

This attempts to upgrade packages already installed in the image when newer candidate versions are available.

It relies on the package catalogue, which is why it is usually preceded by:

```bash
apt-get update
```

## `apt-get dist-upgrade` / `apt-get full-upgrade`

These allow more extensive dependency changes, including installing or removing packages when required to complete an upgrade.

They are more aggressive than normal `upgrade` and are rarely something you should blindly add to an application Dockerfile.

## Summary

```text
apt-get update
    Updates knowledge about available packages

apt-get install curl
    Installs the named package and dependencies

apt-get upgrade
    Upgrades already-installed packages

apt-get full-upgrade
    Performs broader upgrades and dependency changes
```

---

# 8. Why APT commands are combined

A common Dockerfile pattern is:

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
    && rm -rf /var/lib/apt/lists/*
```

This is intentionally one `RUN` instruction.

There are two primary reasons:

1. Docker build caching
2. image-layer cleanup

If you write:

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
```

Docker can cache the `apt-get update` layer independently.

Later, you might change the second line:

```dockerfile
RUN apt-get update
RUN apt-get install -y curl git
```

Docker may reuse the cached package catalogue from the first layer. That catalogue may be stale.

When they are combined:

```dockerfile
RUN apt-get update \
    && apt-get install -y curl git
```

changing the package list invalidates the whole instruction, causing both the catalogue refresh and package installation to run again.

---

# 9. Shell operators used in Dockerfiles

## `&&`

```bash
command1 && command2
```

Run `command2` only if `command1` succeeds.

Example:

```dockerfile
RUN apt-get update \
    && apt-get install -y curl
```

If `apt-get update` fails, installation does not run.

## `;`

```bash
command1 ; command2
```

Run `command2` regardless of whether `command1` succeeded.

This is usually weaker for build operations because failures may be hidden.

## `||`

```bash
command1 || command2
```

Run `command2` only if `command1` fails.

Example:

```bash
test -f /app/config.json || echo "Config missing"
```

Be careful not to use `|| true` merely to suppress legitimate errors.

## `\`

```dockerfile
RUN apt-get update \
    && apt-get install -y \
        curl \
        ca-certificates
```

The backslash continues the instruction onto the next physical line.

## `|`

```bash
command1 | command2
```

Send standard output from `command1` into `command2`.

Example:

```bash
dpkg-query -W | grep ssl
```

## `>`

Overwrite a file:

```bash
echo "hello" > /tmp/message.txt
```

## `>>`

Append to a file:

```bash
echo "world" >> /tmp/message.txt
```

---

# 10. Docker image layers

Docker images are composed of immutable layers.

Consider:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update
RUN apt-get install -y curl
COPY . /app
```

Conceptually:

```text
Layer 1: Ubuntu base filesystem
Layer 2: APT package indexes
Layer 3: curl and dependencies
Layer 4: application files
```

Layers provide:

- reuse between images
- faster downloads
- faster builds
- storage efficiency
- build caching

Two images based on the same Ubuntu image can share the same base layers.

A container adds a writable layer above the image layers:

```text
Container writable layer
Application image layer
Package installation layer
Ubuntu base layers
```

When the container is deleted, its writable layer is normally deleted too.

---

# 11. Docker build caching

Docker tries to reuse the result of previous build steps.

Example:

```dockerfile
COPY MyApp.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release
```

Why copy the project file first?

Because package dependencies usually change less frequently than source code.

When only source code changes:

```text
COPY MyApp.csproj        → cache hit
RUN dotnet restore       → cache hit
COPY . .                 → cache miss
RUN dotnet publish       → rerun
```

This is much faster than copying everything before restore:

```dockerfile
COPY . .
RUN dotnet restore
RUN dotnet publish
```

Any source-file change would then invalidate the earlier `COPY` layer and force restore to run again.

A good general rule:

> Put stable and expensive steps earlier. Put frequently changing files later.

---

# 12. Why cleanup must happen in the same layer

This is weaker:

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

Although the final filesystem view no longer shows the APT indexes, the earlier immutable layer may still contain them.

Conceptually:

```text
Layer 1: add 30 MB package indexes
Layer 2: add curl
Layer 3: mark indexes as deleted
```

The image can still carry the bytes from Layer 1.

This is better:

```dockerfile
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

The indexes are created and deleted before Docker commits the final result of that `RUN` instruction.

Conceptually:

```text
Temporary build container:
    download indexes
    install curl
    delete indexes

Committed layer:
    curl remains
    package indexes do not
```

## `apt-get clean`

You may also see:

```bash
apt-get clean
```

This clears downloaded package archive files from APT's cache.

However, on many official Debian and Ubuntu container images, package archives may already be cleaned or configured not to remain. Removing `/var/lib/apt/lists/*` is still commonly used because `apt-get update` specifically puts package index data there.

---

# 13. Should you run `apt-get upgrade`?

You may see:

```dockerfile
RUN apt-get update \
    && apt-get upgrade -y
```

This is not always wrong, but it should not be added mechanically.

## Potential advantages

- applies currently available package fixes
- may remove some scanner findings
- can be required by an organisation's patch policy

## Potential disadvantages

- makes the image result depend on repository state at build time
- can alter a tested base-image package set
- may produce different images from the same Dockerfile
- can introduce compatibility changes
- may hide the fact that your base image itself is stale
- does not help when no fixed package version exists

A stronger operational model is often:

```text
Base image maintainer publishes patched image
    ↓
Your CI pulls the new base image
    ↓
Your application image is rebuilt
    ↓
Tests and vulnerability scans run
    ↓
New image is deployed
```

Useful build command:

```bash
docker build --pull -t myapp:latest .
```

`--pull` asks Docker to check for a newer version of the base image.

A strict reproducibility strategy may pin images by digest, but then you must deliberately update that digest when patches become available.

## Honest production rule

Do not ask only:

> Should we run `apt-get upgrade`?

Ask:

1. Which vulnerable package is present?
2. Which image layer introduced it?
3. Is it used at runtime?
4. Is a fixed package version available?
5. Is a patched base image available?
6. Can the package be removed?
7. Can the application use a smaller runtime image?
8. Does broad upgrading preserve runtime compatibility?
9. Has the resulting image been tested and rescanned?

---

# 14. Understanding `WORKDIR`, `COPY`, `ARG`, and `ENV`

## `WORKDIR`

```dockerfile
WORKDIR /app
```

This:

- creates `/app` when necessary
- makes it the current working directory
- affects following `RUN`, `COPY`, `CMD`, and `ENTRYPOINT` behavior
- sets the default working directory for the running container

Example:

```dockerfile
WORKDIR /app
COPY MyApp.dll .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

The dot in `COPY MyApp.dll .` means the current working directory, `/app`.

## `COPY`

```dockerfile
COPY . /app
```

The left-hand side comes from the Docker build context.

The right-hand side is inside the image.

```text
Host/build context       Image filesystem
.                 →      /app
```

A Dockerfile cannot normally copy arbitrary files from anywhere on your machine. It can access only the build context and certain explicitly mounted build inputs.

## `ARG`

```dockerfile
ARG BUILD_CONFIGURATION=Release
```

`ARG` defines a build-time variable.

```dockerfile
ARG BUILD_CONFIGURATION=Release

RUN dotnet publish \
    --configuration "$BUILD_CONFIGURATION"
```

Build with:

```bash
docker build \
  --build-arg BUILD_CONFIGURATION=Debug \
  .
```

Do not use ordinary `ARG` or `ENV` instructions for passwords or access tokens. They may leak through image metadata, history, cache, or build logs.

## `ENV`

```dockerfile
ENV ASPNETCORE_HTTP_PORTS=8080
```

This records an environment variable in the image configuration.

A container created from the image receives it by default.

It can be overridden:

```bash
docker run \
  -e ASPNETCORE_HTTP_PORTS=5000 \
  myapp
```

Mental model:

```text
ARG = Docker build input
ENV = image/container environment
```

---

# 15. Understanding `RUN`, `CMD`, and `ENTRYPOINT`

These instructions happen at different times.

## `RUN`

```dockerfile
RUN apt-get update
```

Runs during:

```bash
docker build
```

It changes the image filesystem.

Typical uses:

- install packages
- compile source code
- create users
- change permissions
- download build dependencies

## `CMD`

```dockerfile
CMD ["dotnet", "MyApp.dll"]
```

Defines the default command used when the container starts.

It does not execute during the image build.

A Dockerfile should normally contain only one effective `CMD`. If multiple `CMD` instructions exist, the last one wins.

The command can be replaced:

```bash
docker run myapp dotnet OtherApp.dll
```

## `ENTRYPOINT`

```dockerfile
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

Defines the container's primary executable.

Arguments passed to `docker run` are commonly appended:

```bash
docker run myapp --help
```

Effective command:

```text
dotnet MyApp.dll --help
```

## Combining them

```dockerfile
ENTRYPOINT ["dotnet"]
CMD ["MyApp.dll"]
```

Default effective command:

```text
dotnet MyApp.dll
```

Running:

```bash
docker run myapp OtherApp.dll
```

can replace the `CMD` value, producing:

```text
dotnet OtherApp.dll
```

## Exec form vs shell form

Preferred:

```dockerfile
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

This is exec form.

Less desirable:

```dockerfile
ENTRYPOINT dotnet MyApp.dll
```

This is shell form and often runs through:

```text
/bin/sh -c
```

Exec form generally handles signals and argument boundaries more correctly.

---

# 16. Understanding `EXPOSE`

```dockerfile
EXPOSE 8080
```

This does **not** publish or open the port.

It adds image metadata declaring:

> The application packaged in this image is expected to listen on container port 8080.

It does not:

- make the application listen on port 8080
- publish port 8080 to the host
- create a firewall rule
- configure an AWS security group
- configure an ECS task definition
- create an ALB listener
- configure an ALB target group
- make the container publicly accessible

A .NET example:

```dockerfile
ENV ASPNETCORE_HTTP_PORTS=8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

Here:

```text
ASPNETCORE_HTTP_PORTS=8080
    tells ASP.NET Core to listen on 8080

EXPOSE 8080
    documents the intended container port

ENTRYPOINT
    starts the application
```

The application must actually bind to the expected address and port.

Inside a container, it normally needs to bind to:

```text
0.0.0.0:8080
```

or an equivalent wildcard address.

Binding only to:

```text
127.0.0.1:8080
```

can make the application reachable only from inside that same container.

---

# 17. Port publishing and networking

To publish a container port to your local host:

```bash
docker run -p 5000:8080 myapp
```

This means:

```text
Host port 5000
       ↓
Container port 8080
```

You call:

```text
http://localhost:5000
```

The application inside the container listens on:

```text
0.0.0.0:8080
```

## `-p` versus `-P`

Lowercase:

```bash
docker run -p 5000:8080 myapp
```

Explicit mapping:

```text
host 5000 → container 8080
```

Uppercase:

```bash
docker run -P myapp
```

Publishes all ports declared through `EXPOSE` to automatically selected host ports.

For example:

```text
host 32768 → container 8080
```

Check the selected port:

```bash
docker ps
```

or:

```bash
docker port <container-name>
```

## Publishing to all host interfaces

By default, this can expose the host port on multiple host interfaces:

```bash
docker run -p 5000:8080 myapp
```

For local-only access, explicitly bind to loopback:

```bash
docker run -p 127.0.0.1:5000:8080 myapp
```

This matters because publishing a port can make it reachable beyond your own machine, depending on host networking and firewall configuration.

---

# 18. Who uses `EXPOSE` metadata?

The metadata can be used by several consumers.

## Developers

It documents the expected container port when reading the Dockerfile.

## Image-inspection tools

Inspect it with:

```bash
docker image inspect myapp
```

You may see metadata resembling:

```json
"ExposedPorts": {
  "8080/tcp": {}
}
```

## `docker run -P`

This is the clearest behavior that directly uses exposed-port metadata:

```bash
docker run -P myapp
```

Docker publishes the exposed ports to automatically selected host ports.

## Development and orchestration tooling

Some tools may display, suggest, or infer ports from image metadata.

However, production orchestrators usually require explicit networking configuration.

### Docker Compose

```yaml
services:
  api:
    image: myapp
    ports:
      - "5000:8080"
```

### ECS

You configure a container port mapping in the task definition:

```json
{
  "containerPort": 8080,
  "protocol": "tcp"
}
```

You also separately configure:

- security groups
- target groups
- listeners
- service networking
- Cloud Map or Service Connect when applicable

### Kubernetes

A pod specification can declare:

```yaml
ports:
  - containerPort: 8080
```

A Kubernetes Service is still needed to give stable service networking, and an Ingress or load balancer may be needed for external access.

Therefore:

```text
EXPOSE = image-level documentation and metadata
Routing = runtime/platform configuration
```

---

# 19. Multi-stage builds

A multi-stage Dockerfile uses more than one `FROM`.

Example:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build

WORKDIR /src
COPY . .
RUN dotnet publish \
    --configuration Release \
    --output /out

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime

WORKDIR /app
COPY --from=build /out .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

The build stage contains:

- .NET SDK
- compiler
- NuGet tooling
- source code
- intermediate build output

The final stage contains:

- .NET runtime
- published application
- runtime dependencies

It does not need the complete SDK.

Benefits:

- smaller final image
- fewer packages
- smaller attack surface
- faster image pulls
- fewer scanner findings
- cleaner separation between build and runtime dependencies

Multi-stage builds do not automatically guarantee security. You can still accidentally copy unwanted build files into the runtime stage.

---

# 20. A production-style .NET Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# -------------------------
# Restore and build stage
# -------------------------
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build

ARG BUILD_CONFIGURATION=Release

WORKDIR /src

# Copy project files first for better restore caching.
COPY ["MyApp/MyApp.csproj", "MyApp/"]

RUN dotnet restore "MyApp/MyApp.csproj"

# Copy the remaining source only after restore.
COPY . .

WORKDIR /src/MyApp

RUN dotnet publish "MyApp.csproj" \
    --configuration "$BUILD_CONFIGURATION" \
    --output /app/publish \
    --no-restore \
    /p:UseAppHost=false


# -------------------------
# Runtime stage
# -------------------------
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime

WORKDIR /app

COPY --from=build /app/publish .

ENV ASPNETCORE_HTTP_PORTS=8080

EXPOSE 8080

USER app

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## What happens during build

```text
1. Pull or reuse the .NET SDK image.
2. Copy the project file.
3. Restore NuGet packages.
4. Copy source code.
5. Publish the application.
6. Start a fresh ASP.NET runtime image.
7. Copy only published output.
8. Record runtime configuration.
9. Set the non-root application user.
10. Record the startup executable.
```

## What happens during container startup

```text
1. Create a container from the final image.
2. Attach a writable container layer.
3. Configure networking.
4. Set /app as the working directory.
5. Use the app user.
6. Start: dotnet MyApp.dll.
7. ASP.NET Core listens on port 8080.
```

---

# 21. Common Linux commands in Dockerfiles

## Filesystem navigation

```bash
pwd
```

Print current working directory.

```bash
ls -la
```

List files, including hidden entries, with details.

```bash
cd /app
```

Change directory. In Dockerfiles, prefer `WORKDIR` when the directory should affect subsequent instructions.

## Creating directories

```bash
mkdir /app/data
```

Create one directory.

```bash
mkdir -p /app/data/cache
```

Create parent directories when missing and do not fail merely because an existing directory is present.

## Copying and moving

```bash
cp source destination
mv old-name new-name
```

## Deleting

```bash
rm file.txt
rm -r directory
rm -rf directory
```

`rm -rf` is powerful and dangerous:

- `-r`: recursive
- `-f`: force, without confirmation

Always inspect the path carefully.

## Downloading

```bash
curl -fSL https://example.com/file.tar.gz -o /tmp/file.tar.gz
```

Common flags:

```text
-f  fail on HTTP error responses
-S  show an error message
-L  follow redirects
-o  write output to a named file
```

## Extracting archives

```bash
tar -xzf archive.tar.gz -C /destination
```

Common flags:

```text
-x  extract
-z  gzip compression
-f  archive filename follows
-C  change extraction directory
```

## Searching

```bash
grep "ssl" file.txt
dpkg-query -W | grep -i ssl
find /app -name "*.dll"
```

## Process inspection

```bash
ps aux
```

Minimal images may not include `ps` unless the relevant process tools package is installed.

## Networking

```bash
curl http://localhost:8080/health
getent hosts api
```

Tools such as `ping`, `netstat`, `ss`, `dig`, and `nslookup` may not exist in minimal images.

This is normal. Production images should not automatically contain every debugging tool.

---

# 22. Users, ownership, and permissions

Docker build stages commonly start as root.

That is why this works without `sudo`:

```dockerfile
RUN apt-get update
```

Many minimal images do not contain `sudo`, because it is unnecessary during controlled image construction.

## Why not run the application as root?

A compromised root process inside a container has more capability than a restricted process.

Containers provide isolation, but they are not a magical security boundary.

Use a non-root user when practical:

```dockerfile
USER app
```

or:

```dockerfile
RUN groupadd --system appgroup \
    && useradd --system \
        --gid appgroup \
        --home-dir /app \
        appuser

RUN chown -R appuser:appgroup /app

USER appuser
```

## `chmod`

```bash
chmod +x /app/start.sh
```

Make a script executable.

```bash
chmod 755 /app/start.sh
```

Permissions:

```text
Owner: read, write, execute
Group: read, execute
Others: read, execute
```

## `chown`

```bash
chown appuser:appgroup /app/file
chown -R appuser:appgroup /app
```

Changes ownership.

A useful optimized copy:

```dockerfile
COPY --chown=appuser:appgroup . /app
```

This can avoid adding a separate recursive `chown` layer.

---

# 23. Docker build context and `.dockerignore`

When you run:

```bash
docker build .
```

the final dot means:

> Use the current directory as the build context.

The build context is the set of files Docker makes available to the build.

Example:

```dockerfile
COPY . /src
```

This copies from the build context, not necessarily from the directory containing the Dockerfile.

A large context can:

- slow builds
- invalidate cache unnecessarily
- accidentally include secrets
- copy Git history
- include `bin`, `obj`, or test output
- increase remote-builder upload time

Use `.dockerignore`:

```text
.git
.gitignore
**/bin
**/obj
.vs
.idea
.env
*.user
*.suo
TestResults
coverage
README.md
```

Do not blindly ignore files your build actually needs.

A strong rule:

> Treat the Docker build context as data you are deliberately sending to the builder.

---

# 24. Container filesystem behavior

An image is immutable, but a running container receives a writable layer.

Suppose the image contains:

```text
/app/config.json
```

The application changes it at runtime.

The change exists in that container's writable layer—not in the original image.

If you delete and recreate the container, the change is usually lost.

This is why containers should not be treated like manually maintained servers.

Weak operational model:

```text
Start container
SSH/exec into it
Manually install tools
Modify configuration
Keep it running forever
```

Stronger model:

```text
Change Dockerfile or deployment configuration
Build a new image
Test it
Deploy a new container
Delete the old container
```

This is immutable infrastructure thinking.

---

# 25. Volumes and bind mounts

Persistent or externally managed data should not live only in the container writable layer.

## Named volume

```bash
docker volume create appdata

docker run \
  -v appdata:/app/data \
  myapp
```

Docker manages the host-side storage location.

## Bind mount

```bash
docker run \
  -v "$(pwd)/data:/app/data" \
  myapp
```

This directly maps a host path into the container.

## Difference

```text
Named volume:
    Docker manages storage location

Bind mount:
    You specify a host path
```

In ECS Fargate, persistent storage patterns differ. You may use:

- ephemeral task storage
- EFS for shared persistent files
- S3 for object storage
- RDS or DynamoDB for application state

A container's local filesystem should generally be treated as temporary.

---

# 26. Environment variables and secrets

Environment variables are useful for configuration:

```bash
docker run \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e LOG_LEVEL=Information \
  myapp
```

Do not bake secrets into the image:

```dockerfile
# BAD
ENV DATABASE_PASSWORD=super-secret-password
```

Do not copy secret files into an image:

```dockerfile
# BAD
COPY .env /app/.env
```

Even deleting a secret in a later layer may not remove it from earlier image layers.

For build-time secrets, use BuildKit secret mounts where supported:

```dockerfile
RUN --mount=type=secret,id=nuget_config \
    dotnet restore \
      --configfile /run/secrets/nuget_config
```

At runtime in AWS ECS, use controlled secret injection from services such as:

- AWS Secrets Manager
- AWS Systems Manager Parameter Store

Also apply:

- least-privilege IAM roles
- separate task role and execution role responsibilities
- secret rotation
- logging controls to avoid printing secret values

---

# 27. Signals, PID 1, and graceful shutdown

The main container process usually becomes PID 1 inside the container.

When Docker, ECS, or Kubernetes stops a container, it sends a termination signal.

Your application should:

1. receive the signal
2. stop accepting new work
3. complete or cancel in-flight work appropriately
4. flush necessary state or logs
5. exit within the allowed timeout

Exec form helps signals reach the application directly:

```dockerfile
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

Shell form can introduce an intermediate shell:

```dockerfile
ENTRYPOINT dotnet MyApp.dll
```

That shell may affect signal forwarding.

This matters for:

- ASP.NET Core graceful shutdown
- ECS task replacement
- deployments
- autoscaling
- spot interruption behavior
- long-running background jobs

A container is not simply “killed whenever.” Good applications participate in graceful termination.

---

# 28. Container health checks

A Dockerfile can define:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl --fail http://localhost:8080/health || exit 1
```

But there are important caveats:

- the image must contain `curl`
- adding `curl` only for health checks increases image contents
- ECS health checks and ALB target-group health checks are separate systems
- a Dockerfile health check does not automatically configure an ALB
- overly expensive health checks can hurt the application

For ECS, you may instead configure a task-definition health check using a command available inside the image.

A good health strategy separates:

- **liveness**: is the process fundamentally alive?
- **readiness**: can it receive traffic?
- **dependency status**: are external services available?

Do not make liveness depend on every external dependency. A temporary database issue should not always cause an endless restart loop.

---

# 29. Image tags, digests, and reproducibility

## Tags

```text
ubuntu:24.04
myapp:1.2.3
myapp:latest
```

A tag is a movable name.

The owner can publish a different image under the same tag.

## Digest

```text
ubuntu@sha256:<digest>
```

A digest identifies specific image content.

Pinning by digest gives stronger reproducibility:

```dockerfile
FROM ubuntu:24.04@sha256:<digest>
```

But pinned digests do not update automatically when security patches are released.

You need a controlled dependency-update process.

## Recommended strategy

Use:

- meaningful immutable application version tags
- image digests in deployment systems where appropriate
- automated base-image update checks
- vulnerability scanning
- tested rebuilds
- clear provenance and SBOM generation

Avoid treating `latest` as a reliable production version.

---

# 30. Architectures: `amd64` and `arm64`

Images and native binaries are architecture-specific unless published as multi-platform images.

Common architectures:

```text
linux/amd64
linux/arm64
```

Examples:

- typical Intel/AMD servers: `amd64`
- AWS Graviton: `arm64`
- Apple Silicon: `arm64`

Build multiple platforms:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t example/myapp:1.0 \
  --push \
  .
```

Be careful with:

- native NuGet packages
- manually downloaded binaries
- shell scripts that hardcode `x86_64`
- architecture-specific package URLs
- tests performed only on one architecture

A multi-platform image typically uses a manifest that points clients to the correct architecture-specific image.

---

# 31. Security and vulnerability scanners

A scanner may report packages you never explicitly installed.

They can come from:

1. the Linux base image
2. the .NET runtime image
3. transitive OS dependencies
4. packages installed by your Dockerfile
5. native files copied from the build stage
6. native binaries inside NuGet packages
7. tools accidentally retained in the final image

Example:

```text
Ubuntu layer
    ↓ contains libssl package
.NET runtime layer
    ↓ may rely on native libraries
Your application layer
```

A reported OpenSSL vulnerability may originate from the inherited Linux layers.

Useful inspection commands:

```bash
docker run --rm myimage \
  dpkg-query -W
```

Search package names:

```bash
docker run --rm myimage \
  sh -c 'dpkg-query -W | grep -i ssl'
```

Check shared-library files:

```bash
docker run --rm myimage \
  sh -c 'find /usr/lib /lib -iname "*ssl*" -o -iname "*crypto*" 2>/dev/null'
```

Check OpenSSL CLI version when present:

```bash
docker run --rm myimage \
  openssl version -a
```

Important:

> The absence of the `openssl` executable does not prove that `libssl` or `libcrypto` is absent.

The CLI executable and the shared libraries can be packaged separately.

A scanner finding should be evaluated using:

- package name and version
- image layer or base-image origin
- fixed-version availability
- exploitability and reachability
- runtime usage
- vendor status
- severity and environmental context

Do not dismiss every finding, but do not treat every CVE number as equally exploitable either.

---

# 32. Chiseled and distroless images

## Standard distribution image

A normal Ubuntu or Debian-based runtime image may contain:

- shell
- package manager
- broad OS filesystem
- more troubleshooting utilities
- more packages and metadata

## Chiseled image

Ubuntu chiseled images remove many components not required at runtime.

They commonly omit:

- shell
- package manager
- many command-line utilities
- unnecessary files

Benefits:

- smaller image
- fewer packages
- smaller attack surface
- fewer vulnerability findings

Trade-offs:

- harder interactive debugging
- cannot run `apt-get` in the final image
- scripts requiring `/bin/sh` may fail
- diagnostic commands may not exist
- application requirements must be understood precisely

## Distroless image

Distroless images follow a similar runtime-minimization philosophy: include the application and necessary runtime libraries, but omit a traditional full Linux distribution environment.

A useful distinction:

```text
Standard image:
    application + runtime + broad OS tooling

Chiseled/distroless:
    application + runtime + minimal required OS files
```

These images still use the host Linux kernel.

## Practical multi-stage strategy

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
# Build using full tooling.

FROM <minimal-runtime-image> AS runtime
# Copy only published artifacts.
```

Do not choose a minimal image solely to make scanners look cleaner. Confirm support for:

- globalization/ICU requirements
- timezone data
- native libraries
- certificates
- diagnostics
- CoreWCF dependencies
- cryptography behavior
- filesystem and user requirements

---

# 33. Debugging useful commands

## List images

```bash
docker images
```

## Inspect image metadata

```bash
docker image inspect myapp
```

## View image history

```bash
docker history myapp
```

This helps show Dockerfile-derived layers, although exact detail depends on how the image was built.

## Run an interactive shell

```bash
docker run --rm -it myapp /bin/bash
```

or:

```bash
docker run --rm -it myapp /bin/sh
```

This works only when the image contains the requested shell.

Chiseled and distroless images may contain neither.

## Execute inside a running container

```bash
docker exec -it <container-name> /bin/sh
```

## View logs

```bash
docker logs <container-name>
docker logs -f <container-name>
```

## Inspect processes

```bash
docker top <container-name>
```

## Inspect port mappings

```bash
docker port <container-name>
docker ps
```

## Inspect networking

```bash
docker inspect <container-name>
```

## Copy a file out

```bash
docker cp \
  <container-name>:/app/log.txt \
  ./log.txt
```

## Check disk usage

```bash
docker system df
```

## Remove unused build cache

```bash
docker builder prune
```

Use prune commands carefully. They can remove cached data that would otherwise speed up builds.

---

# 34. Common Docker mistakes

## Mistake 1: Believing `EXPOSE` publishes a port

Wrong:

```text
EXPOSE 8080 means localhost:8080 is available
```

Correct:

```text
EXPOSE records intended container-port metadata
```

You still need:

```bash
docker run -p 5000:8080 myapp
```

or platform-specific routing.

## Mistake 2: Running package installation at container startup

Bad:

```dockerfile
CMD ["sh", "-c", "apt-get update && apt-get install -y curl && ./myapp"]
```

Problems:

- slow startup
- requires root
- depends on external repositories at runtime
- replicas can receive different package versions
- failures become harder to diagnose
- startup behavior becomes non-deterministic

Install dependencies during the image build.

## Mistake 3: Copying everything before dependency restore

Weak:

```dockerfile
COPY . .
RUN dotnet restore
```

Better:

```dockerfile
COPY MyApp.csproj .
RUN dotnet restore
COPY . .
```

This improves cache reuse.

## Mistake 4: Baking secrets into image layers

Bad:

```dockerfile
ARG TOKEN=secret
RUN echo "$TOKEN" > /app/token.txt
RUN rm /app/token.txt
```

The secret may still be recoverable from layers, cache, metadata, or logs.

Use supported build-secret mechanisms.

## Mistake 5: Running as root unnecessarily

Use a non-root runtime user where practical.

## Mistake 6: Installing debugging tools in production images

Installing everything “just in case” increases:

- size
- attack surface
- package count
- vulnerability findings

Prefer separate debugging images, ephemeral debug containers, or controlled diagnostics.

## Mistake 7: Using `latest` as a production version

A mutable tag is not a reliable deployment identity.

## Mistake 8: Treating containers as tiny virtual machines

Containers should usually run one primary service process and be replaced rather than manually repaired.

This does not mean only one operating-system process can ever exist. Runtime helpers and sidecars exist. The real principle is:

> Give each container a clear responsibility and lifecycle.

## Mistake 9: Ignoring graceful shutdown

A process that does not react correctly to termination signals can:

- lose work
- break deployments
- cause requests to fail
- delay ECS task replacement
- corrupt in-progress processing

## Mistake 10: Assuming a smaller image is automatically more secure

Smaller usually helps, but security also depends on:

- package freshness
- runtime configuration
- Linux capabilities
- user privileges
- secrets
- network access
- application vulnerabilities
- supply-chain integrity
- deployment controls

---

# 35. Final mental model

When Docker processes:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY . .

ENV APP_PORT=8080

EXPOSE 8080

USER 10001

ENTRYPOINT ["./myapp"]
```

Think:

```text
1. Start with Ubuntu userspace files.

2. Create a temporary build container.

3. As the current build user:
   - refresh APT package metadata
   - install curl and required dependencies
   - remove temporary APT metadata

4. Commit that final filesystem result as an image layer.

5. Configure /app as the working directory.

6. Copy application files from the build context.

7. Record APP_PORT=8080 as environment configuration.

8. Record that the application expects container port 8080.
   EXPOSE does not publish the port.

9. Configure user 10001 as the runtime user.

10. Record ./myapp as the main startup executable.

11. Produce an immutable image.

12. When docker run is executed:
    - create a writable container layer
    - configure namespaces and networking
    - apply environment variables
    - use /app as the working directory
    - run as user 10001
    - start ./myapp as the main process

13. If -p 5000:8080 is provided:
    - route host port 5000 to container port 8080
```

The most important distinctions are:

```text
Dockerfile
    Recipe for an image

Image
    Immutable packaged filesystem and metadata

Container
    Runtime instance of an image

RUN
    Executes during image build

CMD / ENTRYPOINT
    Executes when the container starts

apt-get update
    Refreshes package metadata

apt-get install
    Installs selected software

apt-get upgrade
    Upgrades already-installed packages

EXPOSE
    Documents an intended container port

docker run -p
    Publishes a container port to a host port
```

---

# 36. Official references

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Docker build best practices](https://docs.docker.com/build/building/best-practices/)
- [Docker build cache](https://docs.docker.com/build/cache/)
- [Optimizing Docker build cache](https://docs.docker.com/build/cache/optimize/)
- [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Understanding image layers](https://docs.docker.com/get-started/docker-concepts/building-images/understanding-image-layers/)
- [Publishing and exposing ports](https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/)
- [Running containers](https://docs.docker.com/engine/containers/run/)
- [Docker build context](https://docs.docker.com/build/concepts/context/)
- [Docker build variables](https://docs.docker.com/build/building/variables/)
- [Docker build secrets](https://docs.docker.com/build/building/secrets/)
- [Docker rootless mode](https://docs.docker.com/engine/security/rootless/)
- [Docker multi-platform builds](https://docs.docker.com/build/building/multi-platform/)

---

## Suggested hands-on exercise

Create this Dockerfile:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        python3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN printf 'from http.server import HTTPServer, SimpleHTTPRequestHandler\nHTTPServer(("0.0.0.0", 8080), SimpleHTTPRequestHandler).serve_forever()\n' \
    > server.py

EXPOSE 8080

CMD ["python3", "server.py"]
```

Build it:

```bash
docker build -t docker-learning .
```

Run without publishing:

```bash
docker run --rm --name docker-learning docker-learning
```

The application listens inside the container, but it is not published to your host.

Run with explicit publishing:

```bash
docker run --rm \
  --name docker-learning \
  -p 5000:8080 \
  docker-learning
```

Open:

```text
http://localhost:5000
```

Try automatic publishing through `EXPOSE`:

```bash
docker run --rm \
  --name docker-learning \
  -P \
  docker-learning
```

In another terminal:

```bash
docker port docker-learning
```

This exercise demonstrates the precise difference between:

```text
application listening
EXPOSE metadata
explicit port publishing
automatic port publishing
```
