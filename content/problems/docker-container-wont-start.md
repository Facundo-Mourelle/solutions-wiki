---
title: "Docker Container Won't Start - Port Already in Use"
date: 2026-08-05T10:00:00Z
draft: false
description: "Fixing docker container startup failures due to port conflicts"
categories: ["DevOps"]
tags: ["docker", "containers", "networking"]
---

## Problem

Docker container fails to start with error:

```
Error response from daemon: driver failed programming external connectivity on container: Bind for 0.0.0.0:8080 failed: port is already allocated
```

## Root Cause

Another process (or a previous container) is already using port 8080. Docker cannot bind to a port that's already in use by the host system or another container.

## Attempted Solutions

- **Attempt 1** — Restarted Docker daemon. Did not work because the port conflict persists at the OS level.
- **Attempt 2** — Tried different port with `-p 8081:80`. Worked but didn't solve the underlying issue.

## Final Solution

```bash
# Find what's using the port
sudo lsof -i :8080

# Kill the process (if safe to do so)
sudo kill -9 <PID>

# OR stop all containers using that port
docker stop $(docker ps -q --filter publish=8080)

# Now start your container
docker run -d -p 8080:80 myapp
```

## Why It Works

The `lsof` command reveals exactly which process holds the port. Killing it frees the port for Docker to bind. Alternatively, stopping orphaned containers releases their port bindings.
