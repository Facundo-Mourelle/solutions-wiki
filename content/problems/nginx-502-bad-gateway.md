---
title: "Nginx 502 Bad Gateway - Upstream Connection Refused"
date: 2026-08-05T12:00:00Z
draft: false
description: "Debugging and fixing nginx 502 bad gateway errors"
categories: ["System Administration"]
tags: ["nginx", "web-server", "networking"]
---

## Problem

Nginx returns 502 Bad Gateway when accessing the application:

```
502 Bad Gateway
nginx/1.24.0
```

Error log shows:

```
connect() failed (111: Connection refused) while connecting to upstream
```

## Root Cause

Nginx cannot connect to the upstream application server (e.g., Node.js, Python, PHP-FPM) because:
1. The upstream service is not running
2. The upstream service is listening on a different port/socket
3. Firewall blocking the connection

## Attempted Solutions

- **Attempt 1** — Restarted nginx. Problem persisted because upstream service was still down.
- **Attempt 2** — Changed proxy_pass to 127.0.0.1. Still failed because upstream wasn't running.

## Final Solution

```bash
# Check if upstream service is running
systemctl status myapp
# OR
ps aux | grep node

# Start/restart the upstream service
sudo systemctl start myapp

# Verify it's listening on expected port
ss -tlnp | grep :3000

# Test nginx config
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

## Why It Works

502 means nginx received an invalid response from upstream. Ensuring the upstream service is running and listening on the correct port allows nginx to proxy requests successfully.
