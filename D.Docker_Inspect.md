#options 
# docker inspect
is used to retrieve **detailed, low-level information** about Docker objects.

It is especially useful when you need to find information that normal commands such as `docker ps` do not show, such as:

- Container IP addresses
    
- Network configuration
    
- Mounts and volumes
    
- Environment variables
    
- Container startup configuration
    
- Image information
    
- Restart policy
    
- Resource configuration
    
- Container state
    
- Container IDs
    
- Port mappings
    
- Docker-created metadata
    

---

# Contents

- [[#Basic Syntax]]
    
- [[#What docker inspect Returns]]
    
- [[#Inspecting Containers]]
    
- [[#Inspecting Images]]
    
- [[#Inspecting Multiple Objects]]
    
- [[#Reading the Output]]
    
- [[#Container State]]
    
- [[#Network Information]]
    
- [[#Mount Information]]
    
- [[#Environment Variables]]
    
- [[#Port Information]]
    
- [[#Startup Configuration]]
    
- [[#Restart Policy]]
    
- [[#Using Go Templates]]
    
- [[#Common Template Examples]]
    
- [[#Inspecting Other Docker Objects]]
    
- [[#Common Mistakes]]
    
- [[#Practical Examples]]
    
- [[#Quick Reference]]
    
- [[#Key Concepts]]
    

---

# Basic Syntax

```bash
docker inspect <object>
```

Example:

```bash
docker inspect my-container
```

Docker returns detailed information about the object in **JSON format**.

The object can be a:

- Container
    
- Image
    
- Network
    
- Volume
    
- Other Docker objects supported by `docker inspect`
    

---

# What docker inspect Returns

Unlike commands such as:

```bash
docker ps
```

which show a short summary, `docker inspect` exposes the object's configuration and metadata.

For example:

```bash
docker inspect my-container
```

may contain information about:

```text
Id
Created
Path
Args
State
Config
Image
NetworkSettings
Mounts
HostConfig
```

The exact fields depend on the type of object being inspected.

### Important

Do not try to memorize the entire JSON structure.

The useful skill is knowing:

1. What information you need.
    
2. Where that information is located.
    
3. How to extract only the field you want.
    

---

# Inspecting Containers

## Basic Container Inspection

```bash
docker inspect <container>
```

Example:

```bash
docker inspect my-container
```

This displays the complete configuration and current state of the container.

You can use either the container name or ID:

```bash
docker inspect my-container
```

or:

```bash
docker inspect 8a4f2c91d123
```

---

# Inspecting Images

`docker inspect` can also inspect images.

```bash
docker inspect <image>
```

Example:

```bash
docker inspect ubuntu
```

Or a specific tag:

```bash
docker inspect ubuntu:24.04
```

Image inspection provides information such as:

- Image ID
    
- Creation time
    
- Architecture
    
- OS
    
- Environment variables
    
- Entrypoint
    
- Default command
    
- Layers
    
- Metadata
    

---

# Inspecting Multiple Objects

You can provide multiple objects:

```bash
docker inspect <object1> <object2>
```

Example:

```bash
docker inspect container1 container2
```

Docker returns information for each object.

You can also inspect different object types when Docker can resolve them:

```bash
docker inspect my-container ubuntu:24.04
```

---

# Reading the Output

A normal:

```bash
docker inspect my-container
```

can produce a very large amount of JSON.

For example, you may see:

```json
[
    {
        "Id": "abc123...",
        "Created": "2026-09-20T10:00:00Z",
        "State": {
            "Status": "running",
            "Running": true
        }
    }
]
```

The output is an array because Docker can inspect multiple objects at once.

Inside it is the actual object information.

---

# Container State

One of the most useful sections is:

```text
State
```

It contains information about the current execution state of the container.

Typical fields include:

```text
Status
Running
Paused
Restarting
OOMKilled
Dead
Pid
ExitCode
Error
StartedAt
FinishedAt
```

Example:

```json
"State": {
    "Status": "running",
    "Running": true,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 12345,
    "ExitCode": 0
}
```

### Useful information

#### Status

```text
running
```

or:

```text
exited
```

#### Running

```text
true
```

or:

```text
false
```

#### ExitCode

Shows the exit status of the container's main process.

For example:

```text
ExitCode: 0
```

usually means the process exited successfully.

A non-zero exit code indicates that the process exited with an error or another non-success status.

#### OOMKilled

```text
true
```

indicates that the container was killed because of an out-of-memory condition.

---

# Network Information

Container networking information is stored under:

```text
NetworkSettings
```

This can contain:

- IP address
    
- Gateway
    
- MAC address
    
- Connected networks
    
- Port mappings
    
- Network aliases
    

For example:

```json
"NetworkSettings": {
    "IPAddress": "172.17.0.2",
    "Gateway": "172.17.0.1"
}
```

You can use this when troubleshooting container networking.

---

# Mount Information

Container mounts are available under:

```text
Mounts
```

This is useful for determining what storage has been attached to a container.

Example information can include:

```text
Type
Name
Source
Destination
Mode
RW
```

For example:

```json
{
    "Type": "bind",
    "Source": "/home/user/project",
    "Destination": "/app",
    "RW": true
}
```

This tells you that:

```text
Host:
 /home/user/project

Container:
 /app
```

are connected using a bind mount.

### Why this is useful

If you forget how a container was started, `docker inspect` can tell you which mounts are currently attached.

---

# Environment Variables

Container environment variables are available under:

```text
Config.Env
```

Example:

```text
"Env": [
    "APP_ENV=production",
    "PORT=8080"
]
```

This can help determine which environment variables were configured when the container was created.

### Important security note

Be careful when inspecting containers.

Environment variables can contain sensitive information such as:

```text
API keys
Passwords
Tokens
Database credentials
```

Do not blindly share the complete output of:

```bash
docker inspect <container>
```

if the container may contain secrets.

---

# Port Information

Port configuration can be found in the inspection output.

For example, a container started with:

```bash
docker run -p 8080:80 nginx
```

has a relationship between:

```text
Host port:       8080
Container port:    80
```

Inspecting the container can help verify that the port publishing configuration exists.

However, remember that `docker ps` is usually much easier when you simply want to see published ports:

```bash
docker ps
```

Use `docker inspect` when you need the detailed configuration behind the mapping.

---

# Startup Configuration

The container's startup configuration can be found mainly under:

```text
Config
```

Important fields include:

```text
Image
Entrypoint
Cmd
WorkingDir
User
Env
ExposedPorts
```

For example:

```json
"Config": {
    "Image": "ubuntu:24.04",
    "WorkingDir": "/app",
    "Cmd": [
        "./script.sh"
    ]
}
```

This can tell you what image the container was created from and what command Docker is configured to run.

---

# Entrypoint vs Cmd

This is particularly useful when troubleshooting containers.

You may see:

```text
Entrypoint
```

and:

```text
Cmd
```

These correspond to the container's configured startup behavior.

For example:

```dockerfile
ENTRYPOINT ["python3"]
CMD ["app.py"]
```

results conceptually in:

```text
python3 app.py
```

Inspecting the container lets you see these values after the container has been created.

---

# Restart Policy

Restart configuration can be found under:

```text
HostConfig.RestartPolicy
```

For example, a container started with:

```bash
docker run --restart unless-stopped nginx
```

will have restart-policy information in its inspection data.

This is useful when troubleshooting containers that:

- Keep restarting
    
- Automatically start again
    
- Do not restart when expected
    

---

# Using Go Templates

The full JSON output is useful for exploration, but it becomes inconvenient when you only need one value.

Docker supports Go templates through:

```bash
--format
```

Syntax:

```bash
docker inspect --format '<template>' <object>
```

Example:

```bash
docker inspect --format '{{.State.Status}}' my-container
```

Instead of printing the entire JSON document, Docker prints:

```text
running
```

This is one of the most useful features of `docker inspect`.

---

# Common Template Examples

## Container Status

```bash
docker inspect --format '{{.State.Status}}' my-container
```

Example output:

```text
running
```

---

## Container IP Address

```bash
docker inspect --format '{{.NetworkSettings.IPAddress}}' my-container
```

Example:

```text
172.17.0.2
```

### Note

For containers connected to user-defined networks, network-specific information is often found under:

```text
.NetworkSettings.Networks
```

rather than relying on the top-level `IPAddress` field.

---

## Container ID

```bash
docker inspect --format '{{.Id}}' my-container
```

---

## Image Used by Container

```bash
docker inspect --format '{{.Config.Image}}' my-container
```

Example:

```text
ubuntu:24.04
```

---

## Working Directory

```bash
docker inspect --format '{{.Config.WorkingDir}}' my-container
```

---

## User

```bash
docker inspect --format '{{.Config.User}}' my-container
```

---

## Entrypoint

```bash
docker inspect --format '{{json .Config.Entrypoint}}' my-container
```

---

## Command

```bash
docker inspect --format '{{json .Config.Cmd}}' my-container
```

Using `json` is useful for fields that contain arrays because it produces readable JSON instead of Go's default representation.

---

# Inspecting Networks

You can inspect a Docker network directly:

```bash
docker inspect <network>
```

Example:

```bash
docker network inspect bridge
```

This can show:

- Network ID
    
- Driver
    
- Subnet
    
- Gateway
    
- Connected containers
    
- Network configuration
    

For example:

```bash
docker network inspect my-network
```

is useful when debugging communication between containers.

---

# Inspecting Volumes

Volumes can also be inspected:

```bash
docker inspect <volume>
```

Example:

```bash
docker inspect my-volume
```

This can show information such as:

```text
Name
Driver
Mountpoint
CreatedAt
Labels
Options
Scope
```

The `Mountpoint` tells you where Docker stores the volume on the host.

### Important

You normally should **not manually modify files inside Docker's volume mountpoint**.

Use the volume through Docker or attach it to a container instead.

---

# Common Mistakes

## Forgetting the Object Name

Incorrect:

```bash
docker inspect
```

Correct:

```bash
docker inspect my-container
```

---

## Inspecting an Object That Does Not Exist

Example:

```bash
docker inspect does-not-exist
```

Docker will report that it cannot find the object.

Check existing objects with:

```bash
docker ps -a
```

or:

```bash
docker image ls
```

---

## Confusing Inspect With Logs

`docker inspect`:

```bash
docker inspect my-container
```

shows configuration, metadata, and state.

`docker logs`:

```bash
docker logs my-container
```

shows the container process's stdout/stderr output.

They answer different questions.

---

## Dumping Everything When You Only Need One Value

Instead of:

```bash
docker inspect my-container
```

you can use:

```bash
docker inspect --format '{{.State.Status}}' my-container
```

when you only need the state.

---

## Assuming Inspect Shows Application Internals

`docker inspect` shows Docker-level information.

It does **not** automatically show:

- Application source code
    
- Files inside the container
    
- Application logs
    
- Database contents
    
- Processes running inside the container
    

For those, you may need:

```bash
docker exec
```

or:

```bash
docker logs
```

depending on what you are investigating.

---

# Practical Examples

## Check Why a Container Is Not Running

First:

```bash
docker ps -a
```

Find the container.

Then:

```bash
docker inspect my-container
```

Look at:

```text
State
```

particularly:

```text
Status
ExitCode
Error
OOMKilled
```

Then check:

```bash
docker logs my-container
```

This gives you both:

```text
Docker-level state
+
Application output
```

---

## Find a Container's IP

```bash
docker inspect --format '{{.NetworkSettings.IPAddress}}' my-container
```

Useful for quick testing on the default bridge network.

---

## Check What Image a Container Uses

```bash
docker inspect --format '{{.Config.Image}}' my-container
```

Example:

```text
ubuntu:24.04
```

---

## Check the Container's Mounts

```bash
docker inspect my-container
```

Then locate:

```text
Mounts
```

This lets you verify whether the container has:

- Bind mounts
    
- Named volumes
    
- Read-only mounts
    
- Read-write mounts
    

---

## Check Container Restart Configuration

```bash
docker inspect my-container
```

Look for:

```text
HostConfig
    RestartPolicy
```

This is useful when a container unexpectedly restarts.

---

# `docker inspect` vs Other Commands

|Command|Main Purpose|
|---|---|
|`docker ps`|List running containers|
|`docker ps -a`|List all containers|
|`docker logs`|View container output|
|`docker exec`|Execute a command inside a running container|
|`docker inspect`|View detailed Docker metadata/configuration|
|`docker image inspect`|Inspect an image|
|`docker network inspect`|Inspect a network|
|`docker volume inspect`|Inspect a volume|

`docker inspect` is primarily a **diagnostic and information-retrieval command**.

---

# Quick Reference

### Inspect container

```bash
docker inspect <container>
```

### Inspect image

```bash
docker inspect <image>
```

### Inspect network

```bash
docker network inspect <network>
```

### Inspect volume

```bash
docker volume inspect <volume>
```

### Inspect multiple objects

```bash
docker inspect <object1> <object2>
```

### Extract a specific field

```bash
docker inspect --format '{{.State.Status}}' <container>
```

### Container IP

```bash
docker inspect --format '{{.NetworkSettings.IPAddress}}' <container>
```

### Container image

```bash
docker inspect --format '{{.Config.Image}}' <container>
```

### Container ID

```bash
docker inspect --format '{{.Id}}' <container>
```

### Working directory

```bash
docker inspect --format '{{.Config.WorkingDir}}' <container>
```

---

# Key Concepts

### `docker inspect` is for detailed information

```bash
docker inspect <object>
```

gives much more information than commands such as:

```bash
docker ps
```

### You do not need to memorize the JSON

Use the full output when exploring.

Use:

```bash
--format
```

when you already know which field you need.

### Different objects expose different information

You can inspect:

```text
Containers
Images
Networks
Volumes
```

### Inspect is mainly a troubleshooting tool

A common troubleshooting workflow is:

```text
docker ps -a
        ↓
docker inspect
        ↓
docker logs
        ↓
docker exec
```

Each command answers a different question.

### Most important fields to recognize

For containers, become familiar with:

```text
State
Config
NetworkSettings
Mounts
HostConfig
```

You do not need to memorize every field inside them. The important skill is being able to locate and extract the information you need.