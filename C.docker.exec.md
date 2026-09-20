#options 
# docker exec

Reference guide for executing commands inside running Docker containers.

`docker exec` is primarily used for:
- Entering a running container
- Running commands inside a container
- Debugging
- Inspecting files and processes
- Running administrative commands
- Opening interactive shells
---

# Contents
[Basic Syntax](#basic-syntax)

[How docker exec works](#how-docker-exec-works)

[Container requirement](#container-requirement)

[Running a command](#running-a-command)

[Interactive mode](#interactive-mode)

[Interactive shells](#interactive-shells)

[TTY](#tty)

[Running commands in the background](#running-commands-in-the-background)

[Environment variables](#environment-variables)

[Environment files](#environment-files)

[Working directory](#working-directory)

[User](#user)

[Privileged mode](#privileged-mode)

[Detaching](#detaching)

[Practical examples](#practical-examples)

[Common mistakes](#common-mistakes)

[Quick reference](#quick-reference)

---

# Basic Syntax

```Docker
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

Example:

```Docker
docker exec my-container ls
```

The structure is:

```text
docker exec
    ↓
OPTIONS
    ↓
CONTAINER
    ↓
COMMAND
    ↓
ARGUMENTS
```

For example:

```Docker
docker exec my-container ls -la /app
```

means:

```text
Container → my-container
Command   → ls
Arguments → -la /app
```

---

# How docker exec works

`docker exec` starts a **new process** inside an already-running container.

For example:

```Docker
docker run -d --name my-container ubuntu sleep 3600
```

The container is running.

You can then execute:

```Docker
docker exec my-container ls /
```

Docker starts `ls /` inside the existing container.

Conceptually:

```text
Container
│
├── Main process
│
├── Process started by docker exec
│
└── Other processes
```

`docker exec` does **not** create another container.

It creates another process inside the existing container.

> NOTE
> 
> `docker run` → creates a new container.
> 
> `docker exec` → runs a new process inside an existing running container.

---

# Container requirement

The target container must be **running**.

For example:

```Docker
docker exec my-container ls
```

will not work if:

```text
my-container
    ↓
Exited
```

Check container status with:

```Docker
docker container ls
```

or:

```Docker
docker container ls -a
```

If the container is stopped, start it first:

```Docker
docker start my-container
```

Then:

```Docker
docker exec my-container ls
```

---

# Running a command

The simplest use is:

```Docker
docker exec <container> <command>
```

Example:

```Docker
docker exec my-container ls
```

Another:

```Docker
docker exec my-container pwd
```

Another:

```Docker
docker exec my-container whoami
```

You can provide arguments:

```Docker
docker exec my-container ls -la /app
```

The command is executed inside the container, so:

```Docker
docker exec my-container pwd
```

reports the container process's working directory, not the host's.

---

# Interactive mode

## `-i, --interactive`

```Docker
docker exec -i <container> <command>
```

keeps standard input (`STDIN`) open.

This is useful when the command expects input.

Example:

```Docker
docker exec -i my-container cat
```

The command can receive input from your terminal.

However, `-i` alone does not provide a terminal interface.

For a normal interactive shell, use:

```Docker
docker exec -it <container> <shell>
```

---

# TTY

## `-t, --tty`

```Docker
docker exec -t <container> <command>
```

allocates a pseudo-terminal (TTY).

TTY allocation is useful for programs designed to interact with a terminal.

It is commonly combined with `-i`.

```Docker
docker exec -it my-container bash
```

The combination means:

```text
-i
 ↓
Keep STDIN open

-t
 ↓
Allocate a pseudo-terminal
```

Therefore:

```Docker
-it
```

is the standard combination for an interactive terminal session.

---

# Interactive shells

One of the most common uses of `docker exec` is opening a shell inside a running container.

## Bash

```Docker
docker exec -it <container> bash
```

Example:

```Docker
docker exec -it my-container bash
```

You are now interacting with Bash running inside the container.

---

## SH

Not every image contains Bash.

For minimal images, try:

```Docker
docker exec -it <container> sh
```

Example:

```Docker
docker exec -it alpine-container sh
```

This is particularly common with Alpine-based images.

> NOTE
> 
> Bash and `sh` are different shells.
> 
> A container does not necessarily contain Bash just because it is Linux-based.

---

# Running commands without a shell

You do not need to open a shell before running a command.

For example:

```Docker
docker exec my-container cat /app/file.txt
```

is enough to execute `cat`.

You can also run:

```Docker
docker exec my-container ls -la /app
```

This is often preferable for automation because you execute exactly the command you need.

---

# Running shell commands

There is an important difference between:

```Docker
docker exec my-container ls -la
```

and:

```Docker
docker exec my-container bash -c "ls -la && pwd"
```

The second explicitly starts a shell and gives it a command string.

This allows shell features such as:

```text
&&
||
>
>>
|
$
*
```

Example:

```Docker
docker exec my-container bash -c "cd /app && ls -la"
```

Without a shell, Docker does not automatically interpret shell syntax.

For example:

```Docker
docker exec my-container echo hello && echo world
```

is interpreted by your **host shell** first.

If you specifically want the commands to execute inside the container, use a shell:

```Docker
docker exec my-container sh -c "echo hello && echo world"
```

> NOTE
> 
> This distinction becomes important when writing scripts that use `docker exec`.

---

# Running a command in the background

## `-d, --detach`

```Docker
docker exec -d <container> <command>
```

runs the command in detached mode.

Docker starts the process and returns without attaching your terminal to its output.

Example:

```Docker
docker exec -d my-container ./background-task.sh
```

The command continues running inside the container after `docker exec` returns, assuming the process itself remains alive.

---

# Environment variables

## `-e, --env`

```Docker
docker exec -e KEY=value <container> <command>
```

sets an environment variable for the process started by `docker exec`.

Example:

```Docker
docker exec \
-e APP_MODE=debug \
my-container \
env
```

The variable is available to that process.

Another example:

```Docker
docker exec \
-e DEBUG=true \
my-container \
./app
```

> NOTE
> 
> This sets the variable for the process started by `docker exec`.
> 
> It does not permanently modify the container's image or Dockerfile configuration.

---

# Environment files

## `--env-file`

```Docker
docker exec --env-file <file> <container> <command>
```

loads environment variables from a file for the process being executed.

Example:

```Docker
docker exec \
--env-file .env \
my-container \
./app
```

This is useful when a command needs several environment variables.

---

# Working directory

## `-w, --workdir`

```Docker
docker exec -w <directory> <container> <command>
```

sets the working directory for the process.

Example:

```Docker
docker exec \
-w /app \
my-container \
ls
```

The command runs as if its working directory were:

```text
/app
```

This is useful when you want to execute a command from a particular location without first running:

```Docker
cd /app
```

Example:

```Docker
docker exec -w /app my-container python app.py
```

---

# User

## `-u, --user`

```Docker
docker exec -u <user> <container> <command>
```

runs the command as a specified user.

Example:

```Docker
docker exec \
-u 1000:1000 \
my-container \
whoami
```

The process runs using UID `1000` and GID `1000`.

You can also use a username if that user exists inside the container:

```Docker
docker exec \
-u appuser \
my-container \
whoami
```

### Why use it?

It allows you to test or execute commands under a particular user's permissions.

For example, an application might normally run as:

```text
appuser
```

while you normally enter the container as root.

You can test the application user's permissions with:

```Docker
docker exec -u appuser my-container ls /app
```

---

# Privileged mode

## `--privileged`

```Docker
docker exec --privileged <container> <command>
```

gives the newly executed process extended privileges.

Example:

```Docker
docker exec --privileged my-container some-command
```

This is an advanced troubleshooting/system-administration feature.

It should not be used casually.

> ⚠️ NOTE
> 
> `--privileged` affects the process being executed by `docker exec`.
> 
> It is not a replacement for proper container security configuration.

If a container was originally designed to run with specific capabilities, granting broad privileges just to make a command work should be avoided where possible.

---

# Detaching from an interactive session

When using:

```Docker
docker exec -it my-container bash
```

you are attached to the shell.

Normally, leaving the shell with:

```bash
exit
```

terminates that shell process.

It does **not** normally stop the container itself.

For example:

```text
Container
│
├── Main application
│
└── Bash ← docker exec
       │
       └── exit
```

After `exit`:

```text
Bash → terminated
Container → still running
```

The container only stops when its main process exits.

---

# Practical examples

## Check files

```Docker
docker exec my-container ls -la
```

---

## Check current directory

```Docker
docker exec my-container pwd
```

---

## Check running processes

```Docker
docker exec my-container ps aux
```

Whether `ps aux` works depends on whether the image contains the required `ps` utility.

---

## Read a file

```Docker
docker exec my-container cat /app/file.txt
```

---

## Open Bash

```Docker
docker exec -it my-container bash
```

---

## Open SH

```Docker
docker exec -it my-container sh
```

---

## Run as another user

```Docker
docker exec -u 1000:1000 my-container whoami
```

---

## Run from a specific directory

```Docker
docker exec -w /app my-container ls
```

---

## Run a shell command

```Docker
docker exec my-container sh -c "cd /app && ls -la"
```

---

## Run a background process

```Docker
docker exec -d my-container ./background-task.sh
```

---

## Set an environment variable

```Docker
docker exec \
-e DEBUG=true \
my-container \
./app
```

---

# Common mistakes

## Container is not running

This:

```Docker
docker exec my-container bash
```

requires `my-container` to be running.

Check:

```Docker
docker container ls
```

If it is stopped:

```Docker
docker start my-container
```

---

## Bash does not exist

You may get an error similar to:

```text
exec: "bash": executable file not found
```

Try:

```Docker
docker exec -it my-container sh
```

Minimal images frequently contain `sh` but not Bash.

---

## Forgetting `-it`

This:

```Docker
docker exec my-container bash
```

starts Bash without providing the normal interactive terminal setup.

For an interactive shell, use:

```Docker
docker exec -it my-container bash
```

---

## Expecting `docker exec` to modify the image

Suppose you run:

```Docker
docker exec my-container apt install nginx
```

The package is installed inside that container.

It does **not** modify the original Docker image.

If you remove the container:

```Docker
docker rm my-container
```

the changes made only to that container are lost.

If you want the configuration to become part of a reproducible image, put the required installation steps in a Dockerfile and rebuild the image.

---

## Confusing `docker exec` with `docker attach`

These commands have different purposes.

```Docker
docker exec
```

starts a **new process** inside the container.

```Docker
docker attach
```

attaches to the container's existing main process.

For example:

```text
docker exec
    ↓
NEW process
    ↓
inside container
```

while:

```text
docker attach
    ↓
EXISTING main process
    ↓
container
```

For opening a shell in a running container, `docker exec -it` is usually what you want.

---

# Command structure

Remember:

```Docker
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

The container comes **before** the command.

Correct:

```Docker
docker exec my-container ls -la
```

Here:

```text
Container → my-container
Command   → ls
Arguments → -la
```

Incorrect:

```Docker
docker exec ls my-container
```

Docker interprets:

```text
Container → ls
```

and:

```text
Command → my-container
```

which is not what you intended.

---

# `docker exec` vs `docker run`

This distinction is important.

### `docker run`

```Docker
docker run ubuntu bash
```

creates a **new container** and starts Bash as its main process.

### `docker exec`

```Docker
docker exec -it existing-container bash
```

starts Bash as an **additional process inside an existing running container**.

Conceptually:

```text
docker run
    ↓
IMAGE
    ↓
NEW CONTAINER
    ↓
MAIN PROCESS
```

while:

```text
docker exec
    ↓
EXISTING CONTAINER
    ↓
ADDITIONAL PROCESS
```

---

# Quick Reference

## Basic command

```Docker
docker exec <container> <command>
```

Execute a command inside a running container.

---

## Interactive

```Docker
docker exec -i <container> <command>
```

Keep standard input open.

---

## TTY

```Docker
docker exec -t <container> <command>
```

Allocate a pseudo-terminal.

---

## Interactive shell

```Docker
docker exec -it <container> bash
```

Open an interactive Bash shell.

```Docker
docker exec -it <container> sh
```

Open an interactive `sh` shell.

---

## Detached

```Docker
docker exec -d <container> <command>
```

Run the command without attaching to its output.

---

## Environment

```Docker
docker exec -e KEY=value <container> <command>
```

Set an environment variable for the executed process.

---

## Environment file

```Docker
docker exec --env-file <file> <container> <command>
```

Load environment variables from a file.

---

## Working directory

```Docker
docker exec -w <directory> <container> <command>
```

Set the process working directory.

---

## User

```Docker
docker exec -u <user> <container> <command>
```

Run the command as a specific user.

---

## Privileged

```Docker
docker exec --privileged <container> <command>
```

Give the executed process extended privileges.

---

# Key Concepts

```text
docker exec
     ↓
Existing running container
     ↓
New process
     ↓
Command executes
```

The most important options are:

```text
-i
    Keep STDIN open

-t
    Allocate a TTY

-it
    Interactive terminal

-d
    Detached execution

-e
    Environment variable

--env-file
    Environment file

-w
    Working directory

-u
    User

--privileged
    Extended privileges
```

For everyday Docker work, the command you will probably use most often is:

```Docker
docker exec -it <container> bash
```

or, for minimal images:

```Docker
docker exec -it <container> sh
```
