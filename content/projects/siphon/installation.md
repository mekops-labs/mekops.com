---
title: "Installation & Deployment"
date: 2026-04-06T12:10:00+02:00
weight: 20
description: "How to deploy Siphon via Docker or as a Home Assistant Add-on."
---

Siphon is distributed as a highly optimized, multi-architecture Go binary packaged via `ko`. It can be deployed as an isolated Docker container or as a deeply integrated Home Assistant Add-on.

### Option A: Home Assistant Add-on (Recommended)

Running Siphon directly within Home Assistant OS provides automatic MQTT credential provisioning, Ingress UI access, and local entity polling via the `SUPERVISOR_TOKEN`.

1. Navigate to your Home Assistant Add-on Store.
2. Click the three dots in the top right and select **Repositories**.
3. Add the MekOps Siphon repository URL: https://github.com/mekops-labs/siphon-ha-addon
4. Install Siphon and start the Add-on.
5. Click **Open Web UI** to access the embedded config editor.

### Option B: Standalone Docker (DMZ / Edge)

For deploying Siphon on remote VPS nodes, Raspberry Pis, or isolated DMZ networks.

1. Create a configuration file:

```bash
mkdir -p /opt/siphon/config
touch /opt/siphon/config/config.yaml
```

2. Deploy via Docker Compose:

```yaml
services:
  siphon:
    image: ghcr.io/mekops-labs/siphon:latest
    container_name: siphon
    command: ["/config/config.yaml"]
    volumes:
      - /opt/siphon/config:/config
    environment:
      - TZ=Europe/Warsaw
      # API Keys can be passed as secure environment variables
      - IOT_PLOTTER_API_KEY=your_secret_key
    restart: unless-stopped
```
