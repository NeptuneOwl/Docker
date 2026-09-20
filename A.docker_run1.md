#options 
# A.docker run1

Expanded reference guide for the most commonly used `docker run` options.

This document focuses on options that are useful for everyday Docker usage and important for understanding how containers work.

---

# Contents

- [Basic Syntax](#basic-syntax)
- [How docker run works](#how-docker-run-works)
- [Container Names](#container-names)
- [Detached & Interactive Containers](#detached--interactive-containers)
- [Automatic Container Removal](#automatic-container-removal)
- [Working Directory](#working-directory)
- [Environment Variables](#environment-variables)
- [Port Publishing](#port-publishing)
- [Storage & Mounts](#storage--mounts)
- [Networking](#networking)
- [Restart Policies](#restart-policies)
- [Entrypoint & Commands](#entrypoint--commands)
- [Combining Options](#combining-options)
- [Common Mistakes](#common-mistakes)

---

# Basic Syntax

```Docker
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```
Example:

```Docker
docker run nginx
```

Here:

```text
OPTIONS   → none
IMAGE     → nginx
COMMAND   → image default command
ARGUMENTS → none
```

example2:

```Docker
docker run -d --name my-web nginx
```

Here:

```text
-d
```

runs the container in `detached mode`.

```text
--name my-web
```

gives the container a custom name.

```text
nginx
```

is the image used to create the container.

---

# How docker run works

`docker run` performs two major actions:

1. Creates a **new container** from an image.
2. Starts that newly created container.

For example:

```Docker
docker run ubuntu
```

it creates a new container based on the `ubuntu` image and then starts it.

Every time you run:

```Docker
docker run ubuntu
```

Docker creates another container.

in nutshell:

```Docker
docker run ubuntu
docker run ubuntu
docker run ubuntu
```

creates **three different containers** from the same image.

To start an existing container instead, use:

```Docker
docker start <container>
```

> NOTE
> 
> > `docker run` = create + start a **new** container.
> > 
> > `docker start` = start an **existing** container.

---

# Container Names

## `--name`

```Docker
docker run --name <name> <image>
```

assigns a custom name to the newly created container.

Example:

```Docker
docker run --name my-web nginx
```

The container can then be referenced using:

```Docker
docker start my-web
docker stop my-web
docker logs my-web
docker inspect my-web
```

instead of using its container ID.

If `--name` is not specified, Docker automatically generates a name.

Example:

```text
focused_curie
happy_tesla
frosty_thompson
```

> NOTE
> 
> > Container names must be unique.
> 
> > You cannot have two containers with the same name at the same time.

---

# Detached & Interactive Containers

## `-d, --detach`

```Docker
docker run -d <image>
```

runs the container in **detached mode**.

Instead of attaching your terminal to the container's main process, Docker starts the container in the background and returns the container ID.

Example:

```Docker
docker run -d nginx
```

You can then check the running container with:

```Docker
docker container ls
```

### When to use

Detached mode is commonly used for services that are expected to keep running:

- Web servers
    
- Databases
    
- APIs
    
- Application servers
    

Example:

```Docker
docker run -d --name web nginx
```

---

## `-i, --interactive`

```Docker
docker run -i <image>
```

keeps the container's standard input (`STDIN`) open.

This is useful when the program running inside the container expects input from the user.

---

## `-t, --tty`

```Docker
docker run -t <image>
```

allocates a pseudo-terminal (TTY) for the container.

This makes the container behave more like a normal terminal session.

`-i` and `-t` are commonly used together:

```Docker
docker run -it <image>
```

Example:

```Docker
docker run -it ubuntu bash
```

This starts Ubuntu and gives you an interactive Bash shell.

> NOTE
> 
> > `-i` = keep input open.
> > 
> > `-t` = allocate a terminal.
> > 
> > `-it` = interactive terminal session.

---

# Automatic Container Removal

## `--rm`

```Docker
docker run --rm <image>
```

automatically removes the container when its main process exits.

Example:

```Docker
docker run --rm ubuntu echo "Hello World"
```

The container is created, runs the command, exits, and is automatically removed.

Without `--rm`:

```Docker
docker run ubuntu echo "Hello World"
```

the container remains after it exits and can be seen with:

```Docker
docker container ls -a
```

### When to use

`--rm` is useful for:

- Temporary containers
    
- Testing
    
- One-off commands
    
- Small utilities
    
- Temporary development environments
    

> NOTE
> 
> > `--rm` removes the **container**, not the image.
> 
> > The image remains available for creating another container.

---

# Working Directory

## `-w, --workdir`

```Docker
docker run -w <directory> <image> <command>
```

sets the working directory inside the container before executing the command.

Example:

```Docker
docker run -w /app ubuntu pwd
```

Output:

```text
/app
```

Without `-w`, Docker uses the working directory configured by the image.

If the image's Dockerfile contains:

```Dockerfile
WORKDIR /app
```

then `/app` is already the default working directory.

`-w` can override it when running the container:

```Docker
docker run -w /tmp my-image
```

> NOTE
> 
> > `-w` affects the container's process at runtime.
> 
> > `WORKDIR` is an image configuration instruction used when building the image.

---

# Environment Variables

## `-e, --env`

```Docker
docker run -e KEY=value <image>
```

creates an environment variable inside the container.

Example:

```Docker
docker run -e APP_MODE=development my-app
```

Inside the container:

```bash
echo $APP_MODE
```

would produce:

```text
development
```

Multiple variables can be supplied:

```Docker
docker run \
-e DB_HOST=postgres \
-e DB_USER=admin \
-e DB_NAME=mydb \
my-app
```

### Why environment variables are useful

Applications commonly use environment variables for configuration such as:

```text
DB_HOST
DB_PORT
DB_USER
DB_PASSWORD
API_KEY
APP_ENV
DEBUG
```

This allows the same image to be configured differently without rebuilding it.

For example:

```Docker
docker run -e APP_ENV=development my-app
```

and:

```Docker
docker run -e APP_ENV=production my-app
```

can use the same image with different runtime configuration.

> ⚠️ NOTE
> 
> > Environment variables can contain sensitive information, but `-e` should not automatically be considered a secure secret-management system.
> 
> > Avoid putting sensitive secrets directly into commands when better secret-management mechanisms are available.

---

## `--env-file`

```Docker
docker run --env-file <file> <image>
```

loads environment variables from a file.

Example:

```Docker
docker run --env-file .env my-app
```

Example `.env`:

```text
DB_HOST=postgres
DB_USER=admin
DB_NAME=mydb
APP_ENV=development
```

This is useful when an application requires many environment variables.

Instead of:

```Docker
docker run \
-e DB_HOST=postgres \
-e DB_USER=admin \
-e DB_NAME=mydb \
-e APP_ENV=development \
my-app
```

you can use:

```Docker
docker run --env-file .env my-app
```

---

# Port Publishing

## `-p, --publish`

```Docker
docker run -p <host-port>:<container-port> <image>
```

publishes a port from the container to the host.

Example:

```Docker
docker run -p 8080:80 nginx
```

The relationship is:

```text
HOST                  CONTAINER

localhost:8080  --->  container:80
```

When you access:

```text
localhost:8080
```

Docker forwards the connection to port `80` inside the container.

### Why this is necessary

A container can have its own network namespace and ports.

For example, an application inside a container may listen on:

```text
0.0.0.0:5000
```

That does not automatically mean you can access it through:

```text
HOST:5000
```

You need to publish the port:

```Docker
docker run -p 5000:5000 my-app
```

---

## Different host and container ports

The two ports do not have to be the same.

```Docker
docker run -p 8080:80 nginx
```

means:

```text
Host port       Container port
    8080   --->      80
```

Another example:

```Docker
docker run -p 9000:3000 my-app
```

means:

```text
Host port       Container port
    9000   --->     3000
```

---

## Multiple ports

You can publish multiple ports:

```Docker
docker run \
-p 8080:80 \
-p 8443:443 \
nginx
```

This publishes:

```text
8080 → 80
8443 → 443
```

---

## `-P, --publish-all`

```Docker
docker run -P <image>
```

publishes all ports declared by the image using `EXPOSE`.

Docker automatically assigns host ports.

For example, an image might contain:

```Dockerfile
EXPOSE 80
EXPOSE 443
```

Running:

```Docker
docker run -P nginx
```

causes Docker to assign host ports automatically.

> NOTE
> 
> > `EXPOSE` does **not** itself publish a port.
> 
> > It documents which ports the application expects to use.
> 
> > `-P` publishes those exposed ports using automatically assigned host ports.

---

# Storage & Mounts

Containers have a writable container filesystem layer, but data stored only in that layer belongs to that particular container.

When the container is removed, that writable layer is removed as well.

Docker provides mounts for data that needs to exist outside the container's writable layer.

---

## `-v, --volume`

The `-v` option can be used with both:

- Named volumes
    
- Bind mounts
    

### Named volume

```Docker
docker run -v <volume-name>:<container-path> <image>
```

Example:

```Docker
docker run -v my-data:/app/data my-app
```

Here:

```text
my-data
```

is a Docker-managed volume.

```text
/app/data
```

is the location where the volume appears inside the container.

---

### Bind mount

```Docker
docker run -v <host-path>:<container-path> <image>
```

Example:

```Docker
docker run -v ./data:/app/data my-app
```

This connects:

```text
Host:
./data

        ↓

Container:
/app/data
```

Changes made to the host directory can therefore be seen from inside the container.

---

## `--mount`

```Docker
docker run --mount type=bind,source=<host-path>,destination=<container-path> <image>
```

is a more explicit syntax for mounts.

Example:

```Docker
docker run \
--mount type=bind,source=./image1/file.txt,destination=/app/file.txt \
image1
```

This creates a bind mount:

```text
Host:
./image1/file.txt

        ↓

Container:
/app/file.txt
```

### Bind mount vs `COPY`

A Dockerfile instruction:

```Dockerfile
COPY ./file.txt /app/file.txt
```

copies the file into the image **during the build**.

A bind mount:

```Docker
--mount type=bind,source=./file.txt,destination=/app/file.txt
```

connects the host file to the container **when the container is run**.

Therefore:

```text
COPY
Host file
   ↓
docker build
   ↓
Image
   ↓
Container
```

while:

```text
Bind mount
Host file
   ↕
Container file
```

This means modifying the host file can immediately affect what the container sees through the mount.

> NOTE
> 
> > A bind mount can hide an existing file at the destination.
> 
> > For example, if the image already contains:
> 
> `/app/file.txt`
> 
> and you mount a host file to:
> 
> `/app/file.txt`
> 
> the mounted host file is what the container sees at that path while the mount exists.

---

# Networking

## `--network`

```Docker
docker run --network <network> <image>
```

connects the container to a specified Docker network.

Docker provides several network modes.

Common examples include:

```text
bridge
host
none
```

and user-created networks.

---

## Default bridge network

When no network is specified, Docker normally connects the container to the default `bridge` network.

Example:

```Docker
docker run -d nginx
```

is effectively using Docker's default networking behavior.

---

## User-defined network

You can create a network:

```Docker
docker network create my-network
```

Then connect containers to it:

```Docker
docker run -d --name web --network my-network nginx
```

Another container can join the same network:

```Docker
docker run -it --network my-network ubuntu bash
```

Containers on the same user-defined network can communicate with each other.

Container names can be used for service discovery on user-defined networks.

For example:

```text
web
```

can be used as the hostname for the container named `web`.

> NOTE
> 
> > `--network` controls the container's network attachment.
> 
> > `-p` controls whether a container port is published to the host.
> 
> > They solve different problems.

---

# Restart Policies

## `--restart`

```Docker
docker run --restart=<policy> <image>
```

defines what Docker should do when the container stops or when Docker itself restarts.

Common policies:

```text
no
on-failure
always
unless-stopped
```

---

## `no`

```Docker
docker run --restart=no <image>
```

Default behavior.

Docker does not automatically restart the container.

---

## `on-failure`

```Docker
docker run --restart=on-failure <image>
```

restarts the container if its main process exits with a non-zero exit code.

You can specify a maximum number of retries:

```Docker
docker run --restart=on-failure:5 <image>
```

---

## `always`

```Docker
docker run --restart=always <image>
```

automatically restarts the container when it stops.

---

## `unless-stopped`

```Docker
docker run --restart=unless-stopped <image>
```

automatically restarts the container unless the container has been manually stopped.

This is commonly useful for long-running services.

Example:

```Docker
docker run -d \
--name web \
--restart=unless-stopped \
nginx
```

---

# Entrypoint & Commands

A Docker image can define default execution behavior using:

```Dockerfile
ENTRYPOINT
```

and:

```Dockerfile
CMD
```

`docker run` can override or replace parts of this behavior.

---

## Running a different command

Anything after the image name is treated as the command and its arguments.

Example:

```Docker
docker run ubuntu echo "Hello"
```

Here:

```text
IMAGE   → ubuntu
COMMAND → echo
ARG     → Hello
```

This allows you to use the same image for different purposes.

For example:

```Docker
docker run ubuntu ls
```

and:

```Docker
docker run ubuntu whoami
```

create separate containers and execute different commands.

---

## `--entrypoint`

```Docker
docker run --entrypoint <command> <image>
```

overrides the image's configured `ENTRYPOINT`.

Example:

```Docker
docker run -it --entrypoint /bin/sh ubuntu
```

This is useful when you want to replace the image's normal startup program.

It is also useful for debugging an image whose normal application command is failing.

---

# Combining Options

Docker options can be combined to configure multiple aspects of a container.

Example:

```Docker
docker run -d \
--name web \
-p 8080:80 \
--restart unless-stopped \
nginx
```

This creates and starts a new container that:

```text
-d
```

runs in the background.

```text
--name web
```

gives the container the name `web`.

```text
-p 8080:80
```

publishes host port `8080` to container port `80`.

```text
--restart unless-stopped
```

configures automatic restarting.

```text
nginx
```

is the image.

Another example using a mount:

```Docker
docker run -d \
--name my-app \
-p 8080:5000 \
--mount type=bind,source=./app,destination=/app \
-e APP_ENV=development \
my-app
```

This demonstrates how several independent container configurations can be specified in a single `docker run` command.

---

# Option Ordering

Docker follows:

```Docker
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

Therefore, Docker options should appear **before the image name**.

Correct:

```Docker
docker run --name my-web -d nginx
```

Incorrect:

```Docker
docker run nginx --name my-web -d
```

In the incorrect example, Docker has already encountered:

```text
nginx
```

as the image.

Everything after the image is interpreted as the command and its arguments rather than Docker CLI options.

This distinction caused an important error during practice with `--mount`:

```Docker
docker run image1 --mount ...
```

Docker attempted to execute:

```text
--mount
```

inside the container.

---

# Common Mistakes

## Confusing `run` with `start`

```Docker
docker run nginx
```

creates a new container.

```Docker
docker start nginx-container
```

starts an existing container.

---

## Putting options after the image

Incorrect:

```Docker
docker run nginx -p 8080:80
```

Correct:

```Docker
docker run -p 8080:80 nginx
```

---

## Confusing host and container ports

```Docker
-p 8080:80
```

means:

```text
HOST:      8080
CONTAINER: 80
```

not the other way around.

---

## Confusing `COPY` with a bind mount

```Dockerfile
COPY ./file.txt /app/file.txt
```

copies the file during image building.

```Docker
--mount type=bind,source=./file.txt,destination=/app/file.txt
```

mounts the host file when the container runs.

---

## Using `watch docker run ...`

`watch` repeatedly executes the command.

For example:

```Docker
watch docker run image1
```

does **not** keep one container running.

Instead, it repeatedly executes:

```Docker
docker run image1
```

which creates a new container each time.

For a long-running container, use detached mode:

```Docker
docker run -d <image>
```

---

# Quick Reference

```Docker
docker run <image>
```

Create and start a new container.

```Docker
docker run -d <image>
```

Run in the background.

```Docker
docker run -it <image> <shell>
```

Run an interactive terminal.

```Docker
docker run --name <name> <image>
```

Give the container a custom name.

```Docker
docker run --rm <image>
```

Automatically remove the container after it exits.

```Docker
docker run -w <directory> <image>
```

Set the working directory.

```Docker
docker run -e KEY=value <image>
```

Set an environment variable.

```Docker
docker run --env-file <file> <image>
```

Load environment variables from a file.

```Docker
docker run -p <host-port>:<container-port> <image>
```

Publish a container port.

```Docker
docker run -P <image>
```

Publish exposed ports using automatically assigned host ports.

```Docker
docker run -v <source>:<destination> <image>
```

Create a volume or bind mount using short syntax.

```Docker
docker run --mount type=bind,source=<host>,destination=<container> <image>
```

Create a bind mount using explicit syntax.

```Docker
docker run --network <network> <image>
```

Connect the container to a Docker network.

```Docker
docker run --restart=<policy> <image>
```

Configure automatic container restarting.

```Docker
docker run --entrypoint <command> <image>
```

Override the image's entrypoint.

```Docker
docker run <image> <command> [ARG...]
```

Run a specific command instead of the image's default command.
