#options 

Reference guide for building Docker images with `docker build`.

This document covers the important concepts and options commonly needed when building images, including build context, Dockerfiles, tags, caching, build arguments, platforms, secrets, and BuildKit features.

---

# Contents

[[#Basic Syntax]]
[[#How docker build works]]
[[#Build Context]]
[[#Dockerfile]]
[[#Tags]]
[[#Build Arguments]]
[[#Build Cache]]
[[#No Cache]]
[[#Pulling the Base Image]]
[[#Build Progress Output]]
[[#Target Build Stages]]
[[#Multi-Platform Builds]]
[[#Build Secrets]]
[[#SSH Forwarding]]
[[#BuildKit Mounts]]
[[#Labels]]
[[#Network During Build]]
[[#Resource & Execution Options]]
[[#Common Build Problems]]
[[#Practical Examples]]
[[#Quick Reference]]

---

# Basic Syntax

```Docker
docker build [OPTIONS] PATH | URL | -
```

The most common form is:

```Docker
docker build .
```

The `.` means:

```text
Current directory
```

Docker uses that directory as the **build context**.

If the directory contains:

```text
Dockerfile
file.txt
readfile.sh
```

then:

```Docker
docker build .
```

can use those files during the build.

---

# How docker build works

A simplified build process looks like:


Dockerfile
+
Build Context
↓
docker build
↓
Build process
↓
Image 
↓
yaaaaaayyyyyyyyyy


For example:

```Docker
docker build -t image1 .
```

Docker:

1. Reads the build context.
2. Finds the Dockerfile.
3. Processes the Dockerfile instructions.
4. Executes required build steps.
5. Uses cached layers when possible.
6. Produces a Docker image.

The resulting image can then be used with:

```Docker
docker run image1
```

> NOTE
> 
> `docker build` creates an **image**.
> 
> `docker run` creates and starts a **container** from an image.

---

# Build Context

The final argument to `docker build` specifies the build context.

Example:

```Docker
docker build .
```

The context is the current directory.

You can also specify another directory:

```Docker
docker build ./my-project
```

or:

```Docker
docker build /home/user/my-project
```

The Dockerfile can only access files that are inside the build context.

For example:

```text
my-project/
├── Dockerfile
├── app.py
└── requirements.txt
```

Building:

```Docker
docker build ./my-project
```

allows the Dockerfile to use:

```Dockerfile
COPY app.py /app/
COPY requirements.txt /app/
```

---

## Why build context matters

Consider:

```text
project/
├── Dockerfile
├── app/
│   └── app.py
└── data/
    └── file.txt
```

If you run:

```Docker
docker build .
```

the entire `project/` directory is the context.

The Dockerfile can access files such as:

```text
app/app.py
data/file.txt
```

using `COPY` or `ADD`.

However, a Dockerfile cannot normally do:

```Dockerfile
COPY /home/user/secret.txt /app/
```

if that file is outside the build context.

> NOTE
> 
> Build context is not simply "where the Dockerfile is."
> 
> You can specify the Dockerfile separately from the context.

---

# `.dockerignore`

A `.dockerignore` file controls which files are excluded from the build context.

Example:

```text
.git
__pycache__
*.pyc
.env
node_modules
```

This prevents unnecessary files from being sent to the builder.

It is particularly important for:

- large directories
- Git repositories
- dependency caches
- generated files
- local secrets
- build artifacts

Example:

```text
project/
├── Dockerfile
├── .dockerignore
├── app.py
└── .git/
```

`.dockerignore`:

```text
.git
.env
__pycache__
```

Now those files are excluded from the build context.

> ⚠️ NOTE
> 
> `.dockerignore` helps prevent files from entering the build context.
> 
> It should not be treated as a replacement for proper secret management.

---

# Dockerfile

By default, Docker looks for:

```text
Dockerfile
```

inside the build context.

Example:

```Docker
docker build .
```

uses:

```text
./Dockerfile
```

---

## `-f, --file`

```Docker
docker build -f <Dockerfile> <context>
```

specifies a different Dockerfile.

Example:

```Docker
docker build -f Dockerfile.dev .
```

Another example:

```Docker
docker build \
-f docker/Dockerfile.production \
.
```

This allows a project to have multiple Dockerfiles:

```text
Dockerfile
Dockerfile.dev
Dockerfile.test
Dockerfile.production
```

---

# Tags

## `-t, --tag`

```Docker
docker build -t <name>:<tag> <context>
```

assigns a name and tag to the resulting image.

Example:

```Docker
docker build -t my-app:1.0 .
```

The image is named:

```text
my-app
```

and tagged:

```text
1.0
```

You can then run:

```Docker
docker run my-app:1.0
```

---

## Why tags matter

Tags are commonly used to identify different image versions:

```text
my-app:1.0
my-app:1.1
my-app:2.0
```

You can also use environment-oriented tags:

```text
my-app:dev
my-app:test
my-app:production
```

Or architecture/version combinations depending on your workflow.

> NOTE
> 
> If no tag is specified when referring to an image, Docker normally uses:
> 
> ```text
> latest
> ```
> 
> as the default tag.
> 
> `latest` does not inherently mean "newest image."

---

## Multiple tags

You can tag the same build with multiple names:

```Docker
docker build \
-t my-app:1.0 \
-t my-app:latest \
.
```

Both tags refer to the resulting image.

---

# Build Arguments

## `--build-arg`

```Docker
docker build \
--build-arg NAME=value \
.
```

passes a build-time argument to the Dockerfile.

Dockerfile:

```Dockerfile
ARG APP_VERSION

RUN echo "Building version $APP_VERSION"
```

Build:

```Docker
docker build \
--build-arg APP_VERSION=1.5 \
.
```

The Dockerfile can use:

```text
APP_VERSION=1.5
```

during the build.

---

## Build arguments vs environment variables

`ARG`:

```Dockerfile
ARG VERSION
```

is primarily for the **build process**.

`ENV`:

```Dockerfile
ENV VERSION=1.0
```

sets an environment variable that is part of the image configuration and is available when the container runs.

Conceptually:

```text
ARG
 ↓
Build time
 ↓
Image creation
```

while:

```text
ENV
 ↓
Image
 ↓
Container runtime
```

They can also be combined when appropriate:

```Dockerfile
ARG VERSION
ENV VERSION=$VERSION
```

---

> ⚠️ NOTE
> 
> Do not use `ARG` for secrets such as passwords or API keys.
> 
> Build arguments can become exposed through build metadata/history depending on how they are used.
> 
> Use BuildKit secrets for sensitive build-time information.

---

# Build Cache

Docker can reuse results from previous builds.

Consider:

```Dockerfile
FROM ubuntu:latest

RUN apt update

COPY app.py /app/

RUN python3 /app/app.py
```

If Docker can reuse an earlier layer, it does not need to execute that step again.

This makes repeated builds significantly faster.

---

## Why Dockerfiles are ordered carefully

Consider:

```Dockerfile
COPY . /app

RUN pip install -r requirements.txt
```

If any file copied by:

```Dockerfile
COPY . /app
```

changes, the cache for later instructions may become invalid.

A common optimization is:

```Dockerfile
COPY requirements.txt /app/

RUN pip install -r /app/requirements.txt

COPY . /app
```

Now changing application source code does not necessarily invalidate the dependency-installation layer.

The general principle is:

```text
Stable instructions
        ↓
before
        ↓
Frequently changing instructions
```

This can significantly improve build performance.

---

# `--no-cache`

```Docker
docker build --no-cache .
```

disables the normal build cache.

Docker rebuilds the instructions rather than reusing cached build layers.

Useful when:

- debugging a build
- testing a clean build
- you suspect stale cached results
- dependencies need to be fetched again

Example:

```Docker
docker build --no-cache -t my-app:test .
```

> NOTE
> 
> `--no-cache` does not necessarily mean "download absolutely everything from scratch."
> 
> It primarily disables reuse of existing build cache for Dockerfile steps.

---

# Pulling the Base Image

## `--pull`

```Docker
docker build --pull .
```

forces Docker to attempt to pull a newer version of referenced base images.

For example:

```Dockerfile
FROM ubuntu:latest
```

Without `--pull`, Docker may use an existing local copy of the base image.

With:

```Docker
docker build --pull .
```

Docker checks the registry for an updated base image.

This can be useful when:

- rebuilding security updates
- updating base images
- ensuring the build starts from the current registry version

Example:

```Docker
docker build \
--pull \
-t my-app:latest \
.
```

> NOTE
> 
> A reproducible production build should generally use deliberately selected image versions or digests rather than blindly depending on a moving `latest` tag.

---

# Build Cache Control

## `--no-cache`

Already covered above:

```Docker
docker build --no-cache .
```

forces Dockerfile steps to be rebuilt rather than reused from cache.

A useful combination is:

```Docker
docker build \
--pull \
--no-cache \
-t my-app:test \
.
```

This is useful when you want to test a clean rebuild using an updated base image.

---

# Build Progress Output

## `--progress`

```Docker
docker build --progress=<mode> .
```

controls build output formatting.

Common modes include:

```text
auto
plain
tty
quiet
```

Example:

```Docker
docker build --progress=plain .
```

This produces more traditional, detailed build output.

This is especially useful when debugging builds or when logs need to be captured.

---

## `-q, --quiet`

```Docker
docker build -q .
```

suppresses most build output and prints the resulting image ID.

Example:

```Docker
docker build -q -t my-app .
```

Useful when you only need the resulting image reference and don't need detailed build output.

---

# Target Build Stages

Dockerfiles can contain multiple build stages.

Example:

```Dockerfile
FROM ubuntu AS base

RUN echo "base"

FROM ubuntu AS development

RUN echo "development"

FROM ubuntu AS production

RUN echo "production"
```

## `--target`

```Docker
docker build \
--target <stage> \
.
```

builds up to a specific stage.

Example:

```Docker
docker build \
--target development \
-t my-app:dev \
.
```

This is useful with multi-stage Dockerfiles.

Typical structure:

```text
Build stage
     ↓
Testing stage
     ↓
Production stage
```

You can target an intermediate stage for development or debugging.

---

# Multi-Platform Builds

## `--platform`

```Docker
docker build \
--platform <platform> \
.
```

specifies the target platform.

Example:

```Docker
docker build \
--platform linux/amd64 \
-t my-app:amd64 \
.
```

Another architecture:

```Docker
docker build \
--platform linux/arm64 \
-t my-app:arm64 \
.
```

This becomes important when building images for systems with different CPU architectures.

---

## Multiple platforms

With BuildKit/buildx workflows, you can build for multiple platforms:

```Docker
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t my-app:latest \
.
```

The resulting workflow can produce an image suitable for multiple architectures.

> NOTE
> 
> Multi-platform builds may require emulation or native builders depending on the target architecture and build steps.

---

# Build Secrets

Builds sometimes need temporary access to secrets.

For example:

```text
Private package repository
Private Git repository
API credentials
Package manager authentication
```

Do not simply write:

```Dockerfile
ARG PASSWORD
```

or:

```Dockerfile
ENV PASSWORD=secret
```

for build secrets.

BuildKit provides secret mounts.

Example:

```Docker
docker build \
--secret id=mysecret,src=secret.txt \
.
```

Dockerfile:

```Dockerfile
RUN --mount=type=secret,id=mysecret \
    cat /run/secrets/mysecret
```

The secret is mounted for that build step rather than intentionally baked into the resulting image.

> ⚠️ NOTE
> 
> Secret handling is an advanced BuildKit feature.
> 
> The important concept is:
> 
> ```text
> Secret
>    ↓
> Temporary build mount
>    ↓
> Build step
>    ↓
> Not intentionally stored in final image
> ```

---

# SSH Forwarding

BuildKit can forward an SSH agent into a build step.

This is useful when a build needs access to a private Git repository.

Example:

```Docker
docker build \
--ssh default \
.
```

Dockerfile:

```Dockerfile
RUN --mount=type=ssh \
    git clone git@github.com:private/repository.git
```

The SSH agent is made available to that build step without copying the user's private SSH key into the image.

> NOTE
> 
> This requires BuildKit support and an SSH agent configured on the host.

---

# BuildKit Mounts

BuildKit supports special mounts inside `RUN` instructions.

One example is:

```Dockerfile
RUN --mount=type=bind,...
```

These mounts exist during the build step.

This is different from:

```Docker
docker run --mount ...
```

which creates a mount for a running container.

Conceptually:

```text
docker build
    ↓
RUN --mount
    ↓
temporary build-time mount
```

versus:

```text
docker run
    ↓
--mount
    ↓
runtime container mount
```

This distinction is important.

---

# Build Cache Mounts

BuildKit can also provide persistent cache directories to build steps.

Example:

```Dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

The cache can speed up repeated builds by preserving package-manager cache data between builds.

This is particularly useful for:

- pip
    
- npm
    
- apt
    
- Go
    
- Rust/Cargo
    
- other package managers
    

The cache is for the build process and should not be confused with application data stored in a runtime container.

---

# Labels

## `--label`

```Docker
docker build \
--label key=value \
.
```

adds metadata labels to the image.

Example:

```Docker
docker build \
--label org.example.version=1.0 \
--label org.example.environment=production \
-t my-app:1.0 \
.
```

Labels can later be inspected using:

```Docker
docker image inspect my-app:1.0
```

They can be useful for:

- version information
- ownership
- environment metadata
- automation
- image management

Labels can also be defined inside the Dockerfile using:

```Dockerfile
LABEL key=value
```

---

# Network During Build

## `--network`

```Docker
docker build \
--network=<mode> \
.
```

controls networking available to `RUN` instructions during the build.

Example:

```Docker
docker build \
--network=host \
.
```

This can be relevant when build steps need network access.

For example:

```Dockerfile
RUN apt update
```

normally needs network connectivity to contact package repositories.

> NOTE
> 
> Build networking and runtime container networking are separate concepts.
> 
> ```Docker
> docker build --network ...
> ```
> 
> affects the build process.
> 
> ```Docker
> docker run --network ...
> ```
> 
> affects the running container.

---

# Resource & Execution Options

Most resource control is more commonly associated with:

```Docker
docker run
```

because that controls the running container.

However, Docker build/buildx workflows also have builder-specific configuration for CPU, memory, concurrency, and other resources.

For ordinary Docker learning, you generally do not need to configure these manually.

The important distinction is:

```text
docker build
    ↓
Resources used while creating the image

docker run
    ↓
Resources available to the running container
```

Do not confuse build resource usage with runtime resource limits.

---

# Build Context from STDIN

Docker can receive build context from standard input.

For example:

```Docker
docker build - < Dockerfile
```

This is an advanced workflow and is useful when the Dockerfile is being generated or supplied by another command.

You may also encounter builds using:

```Docker
docker build -f- .
```

where the Dockerfile is supplied through standard input.

This is useful in automation and scripting but is not normally required for everyday Docker usage.

---

# Building from a Git Repository

The build context can also be a remote URL.

For example, Docker can build from a Git repository URL:

```Docker
docker build https://github.com/user/project.git
```

Docker obtains the build context from the specified source.

This is useful in automated build workflows.

> NOTE
> 
> Remote builds depend on network access and the repository being accessible to the builder.

---

# Common Build Problems

## Dockerfile not found

Example:

```text
failed to read dockerfile
```

Check:

```Docker
ls
```

and make sure the Dockerfile exists in the build context.

Or specify it explicitly:

```Docker
docker build -f Dockerfile.dev .
```

---

## File not found during COPY

Example:

```text
COPY failed
```

Check whether the file is actually inside the build context.

For example:

```text
project/
├── Dockerfile
└── file.txt
```

works with:

```Dockerfile
COPY file.txt /app/
```

but a file outside the build context cannot normally be copied into the image.

---

## `.dockerignore` excluded the file

If a file exists but Docker cannot copy it, check:

```text
.dockerignore
```

The file may have been excluded from the build context.

---

## Cache causing unexpected behavior

Try:

```Docker
docker build --no-cache .
```

If the problem disappears, investigate the build cache and Dockerfile instruction ordering.

---

## Base image is outdated

Try:

```Docker
docker build --pull .
```

or:

```Docker
docker build --pull --no-cache .
```

---

# Practical Examples

## Basic build

Project:

```text
image1/
├── Dockerfile
├── file.txt
└── readfile.sh
```

Build:

```Docker
docker build .
```

---

## Build with a tag

```Docker
docker build -t image1:1.0 .
```

Run:

```Docker
docker run image1:1.0
```

---

## Development build

```Docker
docker build \
-f Dockerfile.dev \
-t my-app:dev \
.
```

---

## Production build

```Docker
docker build \
-f Dockerfile.production \
-t my-app:1.0 \
.
```

---

## Clean build

```Docker
docker build \
--pull \
--no-cache \
-t my-app:test \
.
```

This:

```text
--pull
```

checks for an updated base image.

```text
--no-cache
```

prevents reuse of normal build cache.

```text
-t my-app:test
```

names the resulting image.

---

## Multi-stage build

Dockerfile:

```Dockerfile
FROM ubuntu AS builder

RUN echo "Building..."

FROM ubuntu AS production

RUN echo "Production image"
```

Build only the production target:

```Docker
docker build \
--target production \
-t my-app:production \
.
```

---

## Build with an argument

Dockerfile:

```Dockerfile
FROM ubuntu

ARG APP_VERSION

RUN echo "Version: $APP_VERSION"
```

Build:

```Docker
docker build \
--build-arg APP_VERSION=2.0 \
-t my-app:2.0 \
.
```

---

# Quick Reference

## Basic

```Docker
docker build .
```

Build using the current directory as the context.

```Docker
docker build <path>
```

Build using another directory as the context.

```Docker
docker build -f <Dockerfile> .
```

Use a specific Dockerfile.

---

## Images

```Docker
docker build -t <name>:<tag> .
```

Name/tag the resulting image.

```Docker
docker build -t app:1.0 -t app:latest .
```

Apply multiple tags.

---

## Cache

```Docker
docker build --no-cache .
```

Do not reuse normal build cache.

```Docker
docker build --pull .
```

Check for updated base images.

```Docker
docker build --pull --no-cache .
```

Combine both.

---

## Build arguments

```Docker
docker build --build-arg NAME=value .
```

Pass a build-time argument.

---

## Output

```Docker
docker build --progress=plain .
```

Show detailed build output.

```Docker
docker build -q .
```

Show minimal output.

---

## Multi-stage

```Docker
docker build --target <stage> .
```

Build up to a specific Dockerfile stage.

---

## Platforms

```Docker
docker build --platform linux/amd64 .
```

Build for a specific platform.

For multi-platform workflows:

```Docker
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t my-app:latest \
.
```

---

## Build secrets

```Docker
docker build \
--secret id=<name>,src=<file> \
.
```

Provide a BuildKit secret to the build.

---

## SSH

```Docker
docker build --ssh default .
```

Forward an SSH agent to supported build steps.

---

## Labels

```Docker
docker build \
--label key=value \
.
```

Add metadata to the resulting image.

---

# Core Concepts to Remember

```text
docker build
     ↓
Build Context
     +
Dockerfile
     ↓
Build
     ↓
Image
     ↓
docker run
     ↓
Container
```

The most important distinctions are:

```text
Build context
    ↓
Files available to the build
```

```text
Dockerfile
    ↓
Instructions for creating the image
```

```text
Build cache
    ↓
Avoid unnecessary repeated work
```

```text
ARG
    ↓
Build-time configuration
```

```text
ENV
    ↓
Image/runtime environment configuration
```

```text
--secret
    ↓
Temporary sensitive build data
```

```text
--target
    ↓
Select a stage in a multi-stage build
```

```text
--platform
    ↓
Target CPU/OS platform
```

```text
docker build
    ↓
IMAGE

docker run
    ↓
CONTAINER
```