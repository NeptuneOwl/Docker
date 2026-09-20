# Docker Volume

Docker volumes are Docker-managed storage used to **persist data outside a container's writable layer**.

They are especially useful when data needs to survive:

- Container deletion
    
- Container recreation
    
- Image replacement
    
- Application updates
    

Volumes are commonly used for:

- Databases
    
- Application data
    
- Uploaded files
    
- Persistent configuration
    
- Shared data between containers
    

---

# Contents
- [What Is a Docker Volume](#what-is-a-docker-volume)

- [Volume vs Container Storage](#volume-vs-container-storage)

- [Volume vs Bind Mount](#volume-vs-bind-mount)

- [Basic Syntax](#basic-syntax)

- [Creating a Volume](#creating-a-volume)

- [Listing Volumes](#listing-volumes)

- [Inspecting a Volume](#inspecting-a-volume)

- [Using a Volume With docker run](#using-a-volume-with-docker-run)

- [Mounting a Volume to a Container Path](#mounting-a-volume-to-a-container-path)

- [Writing Data to a Volume](#writing-data-to-a-volume)

- [Volume Persistence](#volume-persistence)

- [Sharing a Volume Between Containers](#sharing-a-volume-between-containers)

- [Read-Only Volumes](#read-only-volumes)

- [Anonymous Volumes](#anonymous-volumes)

- [Volume Drivers](#volume-drivers)

- [Volume Labels](#volume-labels)

- [Removing Volumes](#removing-volumes)

- [Pruning Volumes](#pruning-volumes)

- [Common Mistakes](#common-mistakes)

- [Practical Examples](#practical-examples)

- [Quick Reference](#quick-reference)

- [Key Concepts](#key-concepts)
    

---

# What Is a Docker Volume

A Docker volume is a storage location managed by Docker.

Instead of storing important data only inside a container's writable layer, the data can be stored in a volume.

Conceptually:

```text
Container
    |
    | mount
    ↓
Docker Volume
    |
    ↓
Persistent Data
```

The important property is that the volume exists independently from an individual container.

For example:

```bash
docker volume create my-volume
```

creates the volume.

You can then attach it to:

```text
container1
```

Delete `container1`, create:

```text
container2
```

and attach the same volume.

The data remains.

---

# Volume vs Container Storage

Without a volume, a container has its own writable layer.

For example:

```text
Container
└── /app
    └── file.txt
```

If the container is deleted:

```bash
docker rm my-container
```

the data stored only in that container's writable layer is deleted with it.

With a volume:

```text
Container
└── /app
    └── file.txt
          ↓
       Volume
```

the data is stored separately from the container.

Deleting the container does not automatically delete the named volume.

---

# Volume vs Bind Mount

Both volumes and bind mounts allow data to exist outside the container.

### Named volume

```bash
docker volume create my-volume
```

Docker manages the storage location.

Then:

```bash
docker run --mount type=volume,source=my-volume,target=/app my-image
```

### Bind mount

```bash
docker run --mount type=bind,source=/home/user/project,target=/app my-image
```

You explicitly specify a directory or file on the host.

Conceptually:

```text
Named Volume:

Docker
└── Volume
    └── Data


Bind Mount:

Host filesystem
└── /home/user/project
        ↓
     Container
        /app
```

### General distinction

Use a **volume** when you want Docker to manage persistent application data.

Use a **bind mount** when you specifically want a host file or directory exposed to the container.

---

# Basic Syntax

Docker provides a dedicated volume command:

```bash
docker volume <command>
```

Common commands:

```bash
docker volume create
docker volume ls
docker volume inspect
docker volume rm
docker volume prune
```

---

# Creating a Volume

Create a named volume:

```bash
docker volume create <volume>
```

Example:

```bash
docker volume create my-volume
```

Docker creates the volume and returns its name:

```text
my-volume
```

The volume now exists independently of any container.

---

# Listing Volumes

List all Docker volumes:

```bash
docker volume ls
```

Example output:

```text
DRIVER    VOLUME NAME
local     my-volume
local     database-data
```

This is useful for finding existing volumes before attaching or deleting them.

---

# Inspecting a Volume

Use:

```bash
docker volume inspect <volume>
```

Example:

```bash
docker volume inspect my-volume
```

The output contains information such as:

```text
CreatedAt
Driver
Labels
Mountpoint
Name
Options
Scope
```

Example:

```json
[
    {
        "Name": "my-volume",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/my-volume/_data",
        "Scope": "local"
    }
]
```

### Mountpoint

The `Mountpoint` shows where Docker stores the volume's data on the host.

### Important

Do not normally manipulate the files directly through the host mountpoint.

Access the data through a container instead.

---

# Using a Volume With docker run

A volume is not useful to a container until it is mounted.

Example:

```bash
docker run -d \
    --name my-container \
    --mount type=volume,source=my-volume,target=/app \
    ubuntu
```

This connects:

```text
Docker volume:
my-volume

        ↓

Container:
 /app
```

The container can now read and write data under:

```text
/app
```

and that data is stored in the volume.

---

# Using `-v`

Volumes can also be mounted using the shorter `-v` syntax.

Example:

```bash
docker run -d \
    --name my-container \
    -v my-volume:/app \
    ubuntu
```

The format is:

```text
-v <volume>:<container-path>
```

Example:

```text
-v my-volume:/app
```

means:

```text
my-volume
    ↓
/app
```

### `-v` vs `--mount`

Both can mount named volumes.

```bash
-v my-volume:/app
```

and:

```bash
--mount type=volume,source=my-volume,target=/app
```

perform the same basic operation.

`--mount` is more explicit and provides more configuration options.

---

# Mounting a Volume to a Container Path

The container path is where the volume becomes accessible.

For example:

```bash
docker run --mount type=volume,source=my-volume,target=/data ubuntu
```

Inside the container:

```text
/data
```

is backed by:

```text
my-volume
```

You can choose different container paths:

```bash
docker run --mount type=volume,source=my-volume,target=/app/data ubuntu
```

or:

```bash
docker run --mount type=volume,source=my-volume,target=/var/lib/myapp ubuntu
```

The application simply sees a normal directory.

Docker handles the connection between that directory and the volume.

---

# Writing Data to a Volume

Create a volume:

```bash
docker volume create my-volume
```

Run a container with the volume:

```bash
docker run --rm \
    --mount type=volume,source=my-volume,target=/data \
    ubuntu \
    bash -c 'echo "Hello Docker" > /data/file.txt'
```

The container exits and is removed because of:

```bash
--rm
```

However, the volume remains.

Run another container using the same volume:

```bash
docker run --rm \
    --mount type=volume,source=my-volume,target=/data \
    ubuntu \
    cat /data/file.txt
```

Output:

```text
Hello Docker
```

The first container no longer exists.

The data does.

---

# Volume Persistence

This is one of the most important concepts.

Create a volume:

```bash
docker volume create persistent-data
```

Start a container:

```bash
docker run -d \
    --name container1 \
    --mount type=volume,source=persistent-data,target=/data \
    ubuntu \
    sleep infinity
```

Enter the container:

```bash
docker exec -it container1 bash
```

Create a file:

```bash
echo "persistent data" > /data/test.txt
```

Exit:

```bash
exit
```

Delete the container:

```bash
docker rm -f container1
```

Create another container using the same volume:

```bash
docker run --rm \
    --mount type=volume,source=persistent-data,target=/data \
    ubuntu \
    cat /data/test.txt
```

Output:

```text
persistent data
```

The container was deleted.

The volume was not.

---

# Sharing a Volume Between Containers

Multiple containers can mount the same volume.

For example:

```text
             ┌── Container A
             │
Volume ──────┤
             │
             └── Container B
```

Create the volume:

```bash
docker volume create shared-data
```

Container A:

```bash
docker run -d \
    --name container-a \
    --mount type=volume,source=shared-data,target=/data \
    ubuntu \
    sleep infinity
```

Container B:

```bash
docker run -d \
    --name container-b \
    --mount type=volume,source=shared-data,target=/data \
    ubuntu \
    sleep infinity
```

Both containers now have:

```text
/data
```

connected to the same volume.

If Container A creates:

```text
/data/file.txt
```

Container B can access the same file.

### Important

Sharing a volume does not automatically make applications safe to use concurrently.

For example, two applications writing to the same database files simultaneously can cause data corruption if the application/database does not support that arrangement.

---

# Read-Only Volumes

A volume can be mounted read-only.

With `--mount`:

```bash
docker run \
    --mount type=volume,source=my-volume,target=/data,readonly \
    ubuntu
```

or:

```bash
docker run \
    --mount type=volume,source=my-volume,target=/data,ro \
    ubuntu
```

With `-v`:

```bash
docker run \
    -v my-volume:/data:ro \
    ubuntu
```

The container can read the data but cannot modify it through that mount.

This is useful when one container should consume data without being allowed to change it.

---

# Anonymous Volumes

A volume does not always need a manually chosen name.

For example:

```dockerfile
VOLUME /data
```

can cause Docker to create an **anonymous volume** when a container is created.

Anonymous volumes have generated names rather than names chosen by you.

You may see them with:

```bash
docker volume ls
```

Example:

```text
DRIVER    VOLUME NAME
local     8e9d2f3a...
```

### Named vs Anonymous

Named:

```text
my-volume
```

Anonymous:

```text
8e9d2f3a...
```

Named volumes are generally easier to manage because you know what they are called.

---

# Volume Drivers

Volumes use a storage driver.

The default driver is:

```text
local
```

You can create a volume explicitly using it:

```bash
docker volume create --driver local my-volume
```

Then:

```bash
docker volume inspect my-volume
```

will show:

```text
"Driver": "local"
```

Docker also supports external volume drivers through plugins.

These can provide storage backed by systems outside Docker's normal local storage.

For normal local Docker learning and most simple setups, the `local` driver is enough.

---

# Volume Labels

Volumes can have labels attached to them.

Create a volume:

```bash
docker volume create \
    --label project=myapp \
    --label environment=development \
    my-volume
```

Inspect it:

```bash
docker volume inspect my-volume
```

The labels will appear in the output.

Labels are useful for organizing and identifying Docker resources.

---

# Removing Volumes

Remove a specific volume:

```bash
docker volume rm <volume>
```

Example:

```bash
docker volume rm my-volume
```

### Important

Removing a volume deletes the data stored in it.

Unlike deleting a container, deleting the volume itself removes the persistent storage.

Docker may refuse to remove a volume that is currently being used by a container.

Stop/remove the container first if necessary.

---

# Removing Multiple Volumes

You can specify multiple volumes:

```bash
docker volume rm volume1 volume2 volume3
```

Example:

```bash
docker volume rm test-data cache-data old-data
```

---

# Pruning Volumes

Remove unused volumes:

```bash
docker volume prune
```

Docker asks for confirmation before deleting them.

You can bypass the confirmation with:

```bash
docker volume prune -f
```

### Important

Be careful with volume pruning.

Unused does not necessarily mean unwanted.

A volume may not currently be attached to a container but could still contain important data you intend to use later.

---

# Common Mistakes

## Deleting the Container and Expecting the Volume to Be Deleted

Example:

```bash
docker rm my-container
```

does **not** normally delete a named volume.

The volume remains:

```bash
docker volume ls
```

This is one of the main reasons volumes are useful.

---

## Deleting the Volume and Expecting the Data to Remain

The opposite is also important.

```bash
docker volume rm my-volume
```

deletes the volume and its stored data.

The data does not remain simply because the container still exists.

---

## Forgetting to Mount the Volume

Creating a volume:

```bash
docker volume create my-volume
```

does not automatically attach it to containers.

You still need:

```bash
--mount type=volume,source=my-volume,target=/data
```

or:

```bash
-v my-volume:/data
```

---

## Confusing a Volume With a Bind Mount

This:

```bash
-v my-volume:/data
```

uses a **named volume**.

This:

```bash
-v ./data:/data
```

uses a **bind mount** because `./data` refers to a host filesystem path.

---

## Accidentally Hiding Existing Container Files

Suppose the image contains:

```text
/app
├── config.txt
└── program
```

and you mount a volume:

```bash
--mount type=volume,source=my-volume,target=/app
```

The volume is mounted at `/app`.

The mounted volume becomes the filesystem visible at that location.

Docker can also initialize a newly created empty volume with existing files from the image at the mount location in applicable cases.

Do not assume that mounting storage simply "adds another folder layer" on top of the existing directory.

---

# Practical Examples

## Create and Inspect a Volume

```bash
docker volume create app-data
```

```bash
docker volume inspect app-data
```

---

## Run a Container With a Volume

```bash
docker run -d \
    --name app \
    --mount type=volume,source=app-data,target=/data \
    ubuntu \
    sleep infinity
```

---

## Check the Volume

```bash
docker volume ls
```

---

## Enter the Container

```bash
docker exec -it app bash
```

Then:

```bash
ls -la /data
```

Anything stored there is stored in the volume.

---

## Remove the Container

```bash
docker rm -f app
```

The volume still exists:

```bash
docker volume ls
```

---

## Reuse the Volume

```bash
docker run --rm \
    --mount type=volume,source=app-data,target=/data \
    ubuntu \
    ls -la /data
```

The data from the previous container is still available.

---

# Volume Lifecycle

A typical named-volume workflow looks like:

```text
Create volume
     ↓
docker volume create
     ↓
Mount volume
     ↓
docker run --mount ...
     ↓
Write data
     ↓
Container removed
     ↓
Volume remains
     ↓
New container mounts volume
     ↓
Data still exists
```

The volume's lifecycle is independent from the container's lifecycle.

---

# Volume vs Bind Mount Example

## Named Volume

```bash
docker volume create app-data
```

```bash
docker run \
    --mount type=volume,source=app-data,target=/data \
    ubuntu
```

Docker manages the storage location.

---

## Bind Mount

```bash
docker run \
    --mount type=bind,source=./data,target=/data \
    ubuntu
```

You control the host location:

```text
./data
```

The main difference is **who manages the source storage**.

---

# Quick Reference

### Create

```bash
docker volume create <volume>
```

### List

```bash
docker volume ls
```

### Inspect

```bash
docker volume inspect <volume>
```

### Remove

```bash
docker volume rm <volume>
```

### Remove unused volumes

```bash
docker volume prune
```

### Force volume pruning

```bash
docker volume prune -f
```

### Mount with `--mount`

```bash
docker run \
    --mount type=volume,source=<volume>,target=<container-path> \
    <image>
```

### Mount with `-v`

```bash
docker run \
    -v <volume>:<container-path> \
    <image>
```

### Read-only volume

```bash
docker run \
    --mount type=volume,source=<volume>,target=<container-path>,readonly \
    <image>
```

---

# Key Concepts

### A volume is independent storage

```text
Container
    ↓
Volume
```

The volume can survive the container.

### Creating a volume does not mount it

```bash
docker volume create my-volume
```

only creates it.

You still need to attach it to a container.

### Named volumes are Docker-managed

```bash
docker volume create my-volume
```

Docker manages where the volume's data is stored.

### Bind mounts are host-managed

```bash
--mount type=bind,source=./data,target=/data
```

You explicitly choose the host path.

### Containers can share volumes

Multiple containers can mount the same volume.

### Removing a container is not the same as removing its volume

```bash
docker rm <container>
```

does not normally remove a named volume.

```bash
docker volume rm <volume>
```

removes the volume and its stored data.

### Most important commands

```bash
docker volume create
docker volume ls
docker volume inspect
docker volume rm
docker volume prune
```

### Most important mounting syntax

```bash
--mount type=volume,source=<volume>,target=<container-path>
```

The basic relationship is:

```text
Docker Volume
     ↓
Container Path
```

For example:

```text
my-volume
    ↓
/data
```

Anything written to `/data` is stored in the Docker volume.
