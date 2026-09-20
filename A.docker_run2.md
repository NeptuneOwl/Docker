#options 
# A.docker run — Advanced Reference

Advanced reference for `docker run` options used for security, resource control, networking, filesystem restrictions, hardware access, namespaces, process management, and production workloads.

This document assumes you already understand the basic `docker run` syntax, images, containers, ports, environment variables, and mounts.

---

# Contents

* [[#Security & Isolation]]
* [[#User & Permissions]]
* [[#Filesystem Restrictions]]
* [[#Temporary Filesystems]]
* [[#Linux Capabilities]]
* [[#Privileged Containers]]
* [[#Devices]]
* [[#Resource Limits]]
* [[#CPU Control]]
* [[#Memory Control]]
* [[#GPU Access]]
* [[#Advanced Networking]]
* [[#DNS Configuration]]
* [[#Network Hosts]] 
* [[#Namespaces]]
* [[#Process & PID Limits]]
* [[#Health Checks]]
* [[#Logging]]
* [[#Container Initialization]] 
* [[#Signals & Shutdown]]
* [[#Image Platform & Pull Policy]]
* [[#Kernel & System Configuration]]
* [[#Resource Limits with ulimit]]
* [[#Production Example]]
* [[#Security Example]]
---

# Security & Isolation

Containers are isolated from the host using Linux kernel mechanisms such as:

- namespaces
- cgroups
- capabilities
- filesystem isolation
- security profiles

`docker run` provides options for changing how much isolation a container has.

The general rule is:

> Give a container only the access it actually needs.

Avoid using highly privileged options simply because they make something work.

---

# User & Permissions

## `-u, --user`

```Docker
docker run --user <user> <image>
```

specifies which user the main process runs as inside the container.

Example:

```Docker
docker run --user 1000:1000 my-app
```

This means:

```text
UID = 1000
GID = 1000
```

Instead of running as root.

You can also use a username if that user exists inside the image:

```Docker
docker run --user appuser my-app
```

### Why use it?

By default, many images run their application as root.

Running applications as an unprivileged user reduces the potential impact of a container compromise.

Example:

```Docker
docker run -d \
--name api \
--user 1001:1001 \
my-api
```

> NOTE
> 
> The specified UID/GID must have appropriate permissions for files and directories the application needs to access.

---

# Filesystem Restrictions

## `--read-only`

```Docker
docker run --read-only <image>
```

makes the container's root filesystem read-only.

Example:

```Docker
docker run --read-only nginx
```

The application cannot normally write to the container's root filesystem.

This is useful when an application should not modify its image filesystem at runtime.

However, applications may still need writable locations.

You can provide writable storage using a mount:

```Docker
docker run \
--read-only \
-v app-data:/app/data \
my-app
```

The result is conceptually:

```text
Container root filesystem
        ↓
    READ ONLY

/app/data
        ↓
    READ/WRITE
```

### Security benefit

A read-only root filesystem can reduce the places where an attacker or compromised application can write files.

> NOTE
> 
> `--read-only` does not make the entire container completely unable to write.
> 
> Writable mounts can still provide writable locations.

---

# Temporary Filesystems

## `--tmpfs`

```Docker
docker run --tmpfs <container-path> <image>
```

creates a temporary filesystem inside the container.

Example:

```Docker
docker run \
--tmpfs /tmp \
my-app
```

The `/tmp` directory is backed by temporary storage rather than the normal container filesystem.

Data stored there does not persist as normal container data.

A tmpfs can also be configured:

```Docker
docker run \
--tmpfs /tmp:rw,noexec,nosuid \
my-app
```

Common options include:

```text
rw
ro
nosuid
nodev
noexec
size=
```

### When to use

Useful for:

- temporary files
- caches
- sensitive temporary data
- applications requiring writable `/tmp` while using `--read-only`

Example:

```Docker
docker run \
--read-only \
--tmpfs /tmp:rw,noexec,nosuid \
my-app
```

This gives the application a writable temporary directory while keeping the rest of the root filesystem read-only.

---

# Linux Capabilities

Linux capabilities divide traditionally powerful root privileges into smaller permissions.

Docker removes many capabilities from containers by default.

Two important options are:

```Docker
--cap-add
```

and:

```Docker
--cap-drop
```

---

## `--cap-add`

```Docker
docker run --cap-add <capability> <image>
```

adds a Linux capability to the container.

Example:

```Docker
docker run --cap-add NET_ADMIN my-network-tool
```

This gives the container the `NET_ADMIN` capability.

It can be required for applications that need to perform certain network administration operations.

---

## `--cap-drop`

```Docker
docker run --cap-drop <capability> <image>
```

removes a capability.

Example:

```Docker
docker run --cap-drop NET_RAW my-app
```

This can reduce the privileges available to the container.

You can drop all capabilities:

```Docker
docker run --cap-drop ALL my-app
```

and selectively add only what is required:

```Docker
docker run \
--cap-drop ALL \
--cap-add NET_ADMIN \
my-network-tool
```

### Principle

Prefer:

```text
minimum required capabilities
```

over:

```text
all capabilities
```

---

# Privileged Containers

## `--privileged`

```Docker
docker run --privileged <image>
```

gives the container substantially expanded access to the host.

A privileged container receives many additional Linux capabilities and device access and has significantly weaker isolation from the host.

Example:

```Docker
docker run --privileged -it ubuntu bash
```

### Why it exists

Some specialized workloads require capabilities or devices that normal containers do not have.

Examples can include certain:

- low-level system tools
- hardware interaction
- nested virtualization scenarios
- debugging environments

### Security warning

Do **not** use:

```Docker
--privileged
```

as a generic solution for permission errors.

If an application only needs one capability, prefer:

```Docker
--cap-add
```

If it needs one device, prefer:

```Docker
--device
```

The goal is to grant the smallest required privilege.

---

# Devices

## `--device`

```Docker
docker run --device <host-device>:<container-device> <image>
```

passes a host device into the container.

Example:

```Docker
docker run \
--device /dev/ttyUSB0:/dev/ttyUSB0 \
my-app
```

The container can then access the specified device.

This is useful for workloads that need physical hardware such as certain:

- USB serial devices
- hardware interfaces
- specialized peripherals

Device access should be granted only when required.

---

# Resource Limits

Containers can consume host resources.

Without appropriate limits, one container can potentially consume a disproportionate amount of CPU or memory.

Docker provides resource-control options.

---

# CPU Control

## `--cpus`

```Docker
docker run --cpus <number> <image>
```

sets the maximum amount of CPU time the container can use.

Example:

```Docker
docker run --cpus 2 my-app
```

allows the container to use up to approximately two CPUs worth of processing capacity.

This does not mean the container is permanently assigned two physical CPU cores.

It is a CPU usage limit.

Example:

```Docker
docker run \
-d \
--name api \
--cpus 1.5 \
my-api
```

---

## `--cpuset-cpus`

```Docker
docker run --cpuset-cpus <cpus> <image>
```

restricts the container to specific CPU cores.

Example:

```Docker
docker run --cpuset-cpus 0 my-app
```

restricts the container to CPU core `0`.

Multiple CPUs can be specified:

```Docker
docker run --cpuset-cpus 0,2 my-app
```

or a range:

```Docker
docker run --cpuset-cpus 0-3 my-app
```

### Difference

```text
--cpus
```

controls **how much CPU** the container can consume.

```text
--cpuset-cpus
```

controls **which CPU cores** it can run on.

---

## `--cpu-shares`

```Docker
docker run --cpu-shares <value> <image>
```

sets the container's relative CPU weight when CPU resources are contested.

Example:

```Docker
docker run --cpu-shares 512 app1
docker run --cpu-shares 1024 app2
```

When both containers compete for CPU, `app2` has a higher relative weight.

> NOTE
> 
> CPU shares are not the same as a hard CPU limit.
> 
> Use `--cpus` when you need an actual CPU ceiling.

---

# Memory Control

## `-m, --memory`

```Docker
docker run --memory <limit> <image>
```

sets a memory limit.

Example:

```Docker
docker run --memory 512m my-app
```

The container is limited to approximately:

```text
512 MB
```

of memory.

Common suffixes include:

```text
m
g
```

Examples:

```Docker
--memory 256m
--memory 1g
--memory 2g
```

This is especially useful on systems running multiple containers.

---

## `--memory-reservation`

```Docker
docker run --memory-reservation <limit> <image>
```

sets a softer memory limit.

Example:

```Docker
docker run \
--memory 1g \
--memory-reservation 512m \
my-app
```

Conceptually:

```text
Normal target:
512 MB

Hard maximum:
1 GB
```

The reservation is not a replacement for the hard memory limit.

---

## `--memory-swap`

```Docker
docker run --memory-swap <limit> <image>
```

controls the total amount of memory plus swap available to the container.

For example:

```Docker
docker run \
--memory 512m \
--memory-swap 1g \
my-app
```

conceptually provides:

```text
RAM + swap = up to 1 GB
RAM       = up to 512 MB
```

The exact behavior depends on host swap configuration and the value supplied.

> NOTE
> 
> Memory and swap configuration can become platform/kernel dependent. Test limits on the actual host environment where the container will run.

---

# GPU Access

## `--gpus`

```Docker
docker run --gpus <configuration> <image>
```

makes GPU resources available to the container.

Example:

```Docker
docker run --gpus all my-gpu-app
```

This requests access to all available GPUs.

You can also request a specific GPU depending on the runtime configuration.

Example:

```Docker
docker run --gpus '"device=0"' my-gpu-app
```

GPU support depends on the host GPU, drivers, Docker configuration, and the appropriate container runtime/toolkit.

For NVIDIA GPUs, the NVIDIA Container Toolkit is commonly used.

> NOTE
> 
> `--gpus` does not magically install GPU drivers inside the container.
> 
> The host must already be configured to provide GPU access.

---

# Advanced Networking

## `--network host`

```Docker
docker run --network host <image>
```

uses the host's network namespace rather than providing the container with its own normal network namespace.

Example:

```Docker
docker run --network host my-network-app
```

The application can use the host's network interfaces directly.

This can be useful for applications where network performance or direct host networking is required.

> ⚠️ NOTE
> 
> Host networking reduces network isolation.
> 
> It should not be used simply because it is convenient.

---

## `--network none`

```Docker
docker run --network none <image>
```

starts the container without normal network connectivity.

Example:

```Docker
docker run --network none alpine
```

This is useful for workloads that should not communicate over the network.

---

## Custom networks

A user-defined network can be supplied:

```Docker
docker run --network my-network my-app
```

Custom networks are generally preferable to relying on the default bridge for multi-container applications.

For example:

```Docker
docker network create backend
```

Then:

```Docker
docker run -d \
--name database \
--network backend \
postgres
```

and:

```Docker
docker run -d \
--name api \
--network backend \
my-api
```

The containers can communicate through the Docker network.

---

# DNS Configuration

## `--dns`

```Docker
docker run --dns <server> <image>
```

specifies a DNS server for the container.

Example:

```Docker
docker run \
--dns 1.1.1.1 \
my-app
```

Multiple DNS servers can be supplied:

```Docker
docker run \
--dns 1.1.1.1 \
--dns 8.8.8.8 \
my-app
```

---

## `--dns-search`

```Docker
docker run --dns-search <domain> <image>
```

adds a DNS search domain.

Example:

```Docker
docker run \
--dns-search example.local \
my-app
```

---

## `--dns-option`

```Docker
docker run --dns-option <option> <image>
```

sets DNS resolver options.

This is useful for specialized networking environments.

---

# Network Hosts

## `--add-host`

```Docker
docker run --add-host <hostname>:<IP> <image>
```

adds an entry to the container's `/etc/hosts`.

Example:

```Docker
docker run \
--add-host internal-server:192.168.1.50 \
my-app
```

Inside the container:

```text
internal-server
```

resolves to:

```text
192.168.1.50
```

This can be useful when a container needs a static hostname-to-IP mapping.

---

# Namespaces

Linux namespaces provide isolation between processes and system resources.

Docker can configure several namespace-related behaviors.

---

## `--pid`

```Docker
docker run --pid <mode> <image>
```

controls the container's PID namespace.

Normal containers have their own process namespace.

A specialized mode can allow the container to share the host PID namespace:

```Docker
docker run --pid host <image>
```

This allows processes in the container to see host processes.

### Use case

Useful for certain monitoring/debugging tools.

### Security

Sharing the host PID namespace reduces process isolation.

Do not use it unless required.

---

## `--ipc`

```Docker
docker run --ipc <mode> <image>
```

controls the IPC namespace.

Example:

```Docker
docker run --ipc host my-app
```

allows the container to use the host IPC namespace.

This can be required by certain applications that need to share IPC resources with the host or another process environment.

---

## `--uts`

```Docker
docker run --uts <mode> <image>
```

controls the UTS namespace.

For example:

```Docker
docker run --uts host my-app
```

shares the host's UTS namespace.

This affects things such as the system hostname and domain name.

---

## `--cgroupns`

```Docker
docker run --cgroupns <mode> <image>
```

controls the container's cgroup namespace.

Common modes include:

```text
private
host
```

A private cgroup namespace provides isolation from the host's cgroup hierarchy.

Sharing the host cgroup namespace reduces that isolation.

---

# Process & PID Limits

## `--pids-limit`

```Docker
docker run --pids-limit <number> <image>
```

limits the number of processes that can be created inside the container.

Example:

```Docker
docker run --pids-limit 100 my-app
```

This limits the container to approximately 100 processes.

### Security benefit

A process limit can help mitigate certain fork-bomb style resource-exhaustion attacks.

For example:

```text
Application
    ↓
creates processes
    ↓
100 process limit
    ↓
further process creation blocked
```

The appropriate limit depends on the application.

---

# Health Checks

A container being "running" does not necessarily mean the application inside it is healthy.

For example:

```text
Container: RUNNING
Application: BROKEN
```

Docker health checks provide a mechanism for testing application health.

---

## `--health-cmd`

```Docker
docker run \
--health-cmd "<command>" \
<image>
```

specifies the health-check command.

Example:

```Docker
docker run \
--health-cmd "curl -f http://localhost:8080/health || exit 1" \
my-api
```

If the command succeeds, the health check passes.

If it exits with a failure status, the health check fails.

---

## `--health-interval`

```Docker
docker run \
--health-interval=<duration> \
<image>
```

sets how frequently the health check runs.

Example:

```Docker
--health-interval=30s
```

---

## `--health-timeout`

```Docker
docker run \
--health-timeout=<duration> \
<image>
```

sets how long a health-check attempt can run before timing out.

Example:

```Docker
--health-timeout=5s
```

---

## `--health-retries`

```Docker
docker run \
--health-retries=3 \
<image>
```

sets how many consecutive failures are required before Docker marks the container as unhealthy.

---

## Complete health-check example

```Docker
docker run -d \
--name api \
--health-cmd "curl -f http://localhost:8080/health || exit 1" \
--health-interval=30s \
--health-timeout=5s \
--health-retries=3 \
my-api
```

You can then inspect the container to see its health status.

> NOTE
> 
> Health checks can also be defined in the Dockerfile using `HEALTHCHECK`.

---

# Logging

Docker captures the container's standard output and standard error.

The logging driver controls how Docker handles these logs.

---

## `--log-driver`

```Docker
docker run \
--log-driver=<driver> \
<image>
```

specifies the logging driver.

Example:

```Docker
docker run \
--log-driver=json-file \
my-app
```

The exact drivers available depend on the Docker installation and configuration.

---

## `--log-opt`

```Docker
docker run \
--log-driver=<driver> \
--log-opt <key>=<value> \
<image>
```

provides driver-specific configuration.

Example:

```Docker
docker run \
--log-driver=json-file \
--log-opt max-size=10m \
--log-opt max-file=3 \
my-app
```

This can prevent an application's logs from growing without limit.

---

## `--log-driver=none`

```Docker
docker run \
--log-driver=none \
my-app
```

disables Docker's logging for the container.

Use this carefully because logs may be important for troubleshooting and monitoring.

---

# Container Initialization

## `--init`

```Docker
docker run --init <image>
```

runs a small init process as PID 1 inside the container.

Example:

```Docker
docker run --init my-app
```

### Why is PID 1 special?

The first process inside a container has special responsibilities.

It can need to:

- receive signals
    
- reap orphaned child processes
    
- properly handle process termination
    

Some applications do not handle these responsibilities well when they become PID 1.

`--init` provides a small init process to help with this.

This can be useful for applications that spawn child processes.

---

# Signals & Shutdown

## `--stop-signal`

```Docker
docker run \
--stop-signal=<signal> \
<image>
```

changes the signal Docker sends when stopping the container.

Example:

```Docker
docker run \
--stop-signal SIGQUIT \
my-app
```

By default, Docker normally sends `SIGTERM` to the container's main process when stopping it, followed by a forced kill after the configured timeout if it does not exit.

---

## `--stop-timeout`

```Docker
docker run \
--stop-timeout=<seconds> \
<image>
```

sets how long Docker waits for the container to stop before forcefully terminating it.

Example:

```Docker
docker run \
--stop-timeout 30 \
my-app
```

This gives the application more time to perform graceful shutdown tasks.

Useful for applications that need to:

- finish requests
    
- flush buffers
    
- close database connections
    
- save state
    
- finish file operations
    

---

# Image Platform & Pull Policy

## `--platform`

```Docker
docker run --platform <platform> <image>
```

specifies the platform for the image.

Example:

```Docker
docker run --platform linux/amd64 my-app
```

Common platform combinations include:

```text
linux/amd64
linux/arm64
```

This is particularly relevant when running containers on different CPU architectures.

For example:

```text
AMD64 computer
        ↓
linux/amd64

ARM64 computer
        ↓
linux/arm64
```

Multi-platform images can provide different image variants for each architecture.

---

## `--pull`

```Docker
docker run --pull=<policy> <image>
```

controls when Docker pulls the image.

Common policies include:

```text
missing
always
never
```

Example:

```Docker
docker run --pull=always nginx
```

forces Docker to check for a newer image before running.

Example:

```Docker
docker run --pull=never nginx
```

prevents Docker from pulling the image.

This can be useful when controlling image availability and update behavior in controlled environments.

---

# Kernel & System Configuration

## `--sysctl`

```Docker
docker run \
--sysctl <key>=<value> \
<image>
```

configures certain kernel parameters for the container.

Example:

```Docker
docker run \
--sysctl net.ipv4.ip_forward=1 \
my-network-app
```

Only kernel parameters supported by Docker's namespace/security model can be changed this way.

> ⚠️ NOTE
> 
> `--sysctl` is a low-level option.
> 
> Do not modify kernel parameters unless you understand what the parameter controls.

---

# Resource Limits with ulimit

## `--ulimit`

```Docker
docker run \
--ulimit <resource>=<soft>:<hard> \
<image>
```

sets process resource limits.

Example:

```Docker
docker run \
--ulimit nofile=1024:4096 \
my-app
```

This controls the number of file descriptors the process can use.

Conceptually:

```text
Soft limit = 1024
Hard limit = 4096
```

This can be important for applications that handle large numbers of:

- connections
    
- files
    
- sockets
    

The available limits depend on the host kernel and Docker configuration.

---

# Production Example

A more realistic production-style container might look like:

```Docker
docker run -d \
--name production-api \
--restart unless-stopped \
--read-only \
--user 1001:1001 \
--cap-drop ALL \
--tmpfs /tmp:rw,noexec,nosuid \
--memory 1g \
--cpus 2 \
--pids-limit 200 \
--network app-network \
-p 8080:8080 \
--env-file .env.production \
-v api-data:/app/data \
--health-cmd "curl -f http://localhost:8080/health || exit 1" \
--health-interval 30s \
--health-timeout 5s \
--health-retries 3 \
my-company/api:v2.1.0
```

The important concepts are:

```text
--restart
    Automatically restart the service.

--read-only
    Protect the root filesystem from writes.

--user
    Do not run the application as root.

--cap-drop ALL
    Remove unnecessary Linux capabilities.

--tmpfs
    Provide a temporary writable /tmp.

--memory
    Limit memory usage.

--cpus
    Limit CPU usage.

--pids-limit
    Limit process creation.

--network
    Connect to the application network.

-p
    Publish the application port.

--env-file
    Supply runtime configuration.

-v
    Persist application data.

--health-cmd
    Test application health.
```

This demonstrates the principle of **least privilege + resource control + controlled persistence**.

---

# Security Example

A security-conscious container might use:

```Docker
docker run -d \
--name secure-app \
--read-only \
--user 1000:1000 \
--cap-drop ALL \
--tmpfs /tmp:rw,noexec,nosuid \
--pids-limit 100 \
--memory 512m \
--cpus 1 \
--network app-network \
my-app
```

The container receives:

```text
Limited filesystem writes
        +
Non-root user
        +
No unnecessary capabilities
        +
Temporary writable /tmp
        +
Process limit
        +
Memory limit
        +
CPU limit
        +
Controlled network
```

This does not make the application "secure" by itself.

Container security is a layered process involving:

- image security
    
- application security
    
- Linux permissions
    
- capabilities
    
- namespaces
    
- filesystem controls
    
- network isolation
    
- secret management
    
- vulnerability management
    
- runtime monitoring
    

---

# Advanced Option Summary

## Security

```Docker
--user
--read-only
--cap-add
--cap-drop
--privileged
--device
```

## Resources

```Docker
--memory
--memory-reservation
--memory-swap
--cpus
--cpu-shares
--cpuset-cpus
--pids-limit
--ulimit
```

## Networking

```Docker
--network
--dns
--dns-search
--dns-option
--add-host
```

## Filesystem

```Docker
--read-only
--tmpfs
--mount
```

## Namespaces

```Docker
--pid
--ipc
--uts
--cgroupns
```

## Health & Lifecycle

```Docker
--health-cmd
--health-interval
--health-timeout
--health-retries
--init
--stop-signal
--stop-timeout
```

## Logging

```Docker
--log-driver
--log-opt
```

## Platform

```Docker
--platform
--pull
```

## Kernel

```Docker
--sysctl
```

---

# Important Principle

Advanced `docker run` options should generally be used to **reduce or precisely control access**, rather than to grant broad access.

Prefer:

```Docker
--cap-add NET_ADMIN
```

over:

```Docker
--privileged
```

Prefer:

```Docker
--device /dev/...
```

over:

```Docker
--privileged
```

Prefer:

```Docker
--read-only
```

with specific writable mounts over making the entire filesystem writable.

Prefer:

```Docker
--user 1000:1000
```

over running applications as root when possible.

The objective is:

```text
Application
     ↓
minimum privileges
     ↓
minimum resources
     ↓
minimum filesystem access
     ↓
minimum network access
     ↓
controlled container
```