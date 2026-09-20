#options 
# A.docker run — Advanced Reference

Advanced reference for `docker run` options used for security, resource control, networking, filesystem restrictions, hardware access, namespaces, process management, and production workloads.

This document assumes you already understand the basic `docker run` syntax, images, containers, ports, environment variables, and mounts.

---

# Contents

* [Security & Isolation](#security--isolation)
* [User & Permissions](#user--permissions)
* [Filesystem Restrictions](#filesystem-restrictions)
* [Temporary Filesystems](#temporary-filesystems)
* [Linux Capabilities](#linux-capabilities)
* [Privileged Containers](#privileged-containers)
* [Devices](#devices)
* [Resource Limits](#resource-limits)
* [CPU Control](#cpu-control)
* [Memory Control](#memory-control)
* [GPU Access](#gpu-access)
* [Advanced Networking](#advanced-networking)
* [DNS Configuration](#dns-configuration)
* [Network Hosts](#network-hosts)
* [Namespaces](#namespaces)
* [Process & PID Limits](#process--pid-limits)
* [Health Checks](#health-checks)
* [Logging](#logging)
* [Container Initialization](#container-initialization)
* [Signals & Shutdown](#signals--shutdown)
* [Image Platform & Pull Policy](#image-platform--pull-policy)
* [Kernel & System Configuration](#kernel--system-configuration)
* [Resource Limits with ulimit](#resource-limits-with-ulimit)
* [Production Example](#production-example)
* [Security Example](#security-example)
---

# Security & Isolation
