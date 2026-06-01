---
title: "Siphon v0.5.x: Zero-Downtime Hot Reloading and The Garmin-Tailscale Funnel"
author: Kamil Wcisło
date: 2026-04-21
draft: false
toc: true
series: siphon
tags:
  - tailscale
  - garmin
  - home_assistant
  - mqtt
  - security
  - etl
  - iot
thumbnail: images/20260421/siphon_05x.webp
summary: |
    Siphon v0.5.0 introduces the critical new feature for a gateway app: zero-downtime hot reloading.
    A powerful new integration is also showcased — using tailscale funnel and the {API}Call Garmin widget to securely
    bridge a watch to Home Assistant via MQTT, without exposing a single port on my router.
---

> [Siphon's project page](/projects/siphon)

The "Zero-Trust" mantra often comes with a significant usability tax. If you want to trigger a **Home Assistant**
automation from a wearable while you're out on a run, you're usually forced to choose between the security risk of an
open port or the complexity of a VPN that your watch can't natively run. You generally don't want to expose the
whole Home Assistant API to the internet, especially when all you need is to trigger a light from a watch. Principle of
least privilege.

Siphon v0.5.0 solves this. By combining the new **webhook** collector, **hass** sink and **tailscale funnel** you
can build a secure, declarative ingress point that bridges the gap between your wrist and your smart home — with no
custom app development and no open firewall rules.

## Hot reloading: engineering for uptime and ease

In previous versions, updating your `config.yaml` required a manual restart of the Siphon binary or container. In a
distributed system, this "blip" in telemetry ingestion is often unacceptable.

V0.5.0 introduces a true **hot reloading** engine. When you save your configuration via the integrated web editor,
Siphon performs a graceful transition:

1. It validates the new YAML schema.
2. It signals the existing engine to drain active connections.
3. It spawns a new engine instance with the updated pipeline graph.
4. **The HTTP server remains alive throughout**, ensuring no incoming webhooks are dropped during the transition.

![Siphon Web Editor showing the Saved and reloaded status line](images/20260421/editor_reload.webp)

## Pipeline engine changes

Two smaller but load-bearing changes shipped in v0.5.0 that affect how pipelines are written.

### Transform block is now an array

Previously, `transform` was a YAML map. Maps have no guaranteed key order, which meant chained transformations — where one expression depends on the result of a previous one — could silently evaluate in the wrong order depending on the parser implementation.

It is now an **ordered array**:

```yaml
transform:
  - temp_c: float(raw_temp)
  - temp_f: (temp_c * 9/5) + 32       # safe: temp_c is guaranteed to exist
  - label: string(temp_f) + "°F"      # safe: temp_f is guaranteed to exist
```

If you are migrating an existing config, replace the map syntax (`key: value`) with the array syntax (`- key: value`).

### Topic selectors for variables

In **stateful** pipelines that subscribe to multiple upstream topics, each source pipeline's variables are accessible via its name as a namespace. This is what makes the cron-based aggregation pattern work cleanly — one dispatcher can pull the latest cached state from several independent collectors and merge them into a single payload.

In this example, the `file` collector reads the CPU temperature and the `rest` collector (new in v0.5.0) polls an HTTPS
endpoint on an interval and publishes the response body as a raw payload. Combined with a `jsonpath` parser and topic
namespacing, they compose cleanly into the aggregation pattern:

```yaml
collectors:
  weather:
    type: rest
    params:
      interval: 60
    topics:
      weather: "https://api.example.com/weather/current"

  cpu_temp:
    type: file
    params:
      interval: 60
    topics:
      cpu_raw: "/sys/class/thermal/thermal_zone0/temp"

sinks:
  notify:
    type: mqtt
    params:
      url: "tcp://192.168.1.1:1883"

pipelines:
  - name: outdoor_temp
    topics: ["weather"]
    parser:
      type: jsonpath
      vars:
        state: "$.temperature"

  - name: cpu_temp
    topics: ["cpu_raw"]
    parser:
      type: regex
      vars:
        temp_str: "[0-9]+"

  # Merge them using the internal Expr engine
  - name: unified_report
    type: cron
    stateful: true
    schedule: "0 0 * * * *"
    topics: ["outdoor_temp", "cpu_temp"]
    sinks:
      - name: notify
        format: expr
        spec: |
          {
            "out_temp": outdoor_temp?.state ?? 0,
            "cpu_temp": cpu_temp?.temp_str ?? 0
          }
```

The `?.` safe-navigation operator prevents panics when a source pipeline hasn't yet received data since startup.

> For more details, see [configuration](/projects/siphon/configuration) project page.

## The Garmin-Tailscale Funnel pattern

The highlight of this release is the ingress pattern. A Garmin watch triggers a local Home Assistant automation with a
single button press — over the public internet, fully encrypted, zero ports opened.

### The full stack

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'edgeLabelBackground':'#1a1a1a', 'lineColor': '#94a3b8', 'textColor': '#94a3b8'}}}%%
flowchart LR
    classDef external fill:#2a1215,stroke:#7f1d1d,stroke-width:1px,color:#ef4444
    classDef tunnel  fill:#0f172a,stroke:#2563eb,stroke-width:1px,color:#60a5fa
    classDef siphon  fill:#281809,stroke:#b45309,stroke-width:1px,color:#fbbf24
    classDef ha      fill:#064e3b,stroke:#059669,stroke-width:1px,color:#34d399

    A["⌚<br/>Garmin<br/>watch"]:::external
    B["🌐<br/>tailscale<br/>funnel"]:::tunnel
    C["⚙️<br/>webhook<br/>collector"]:::siphon
    D["🔄<br/>pipeline"]:::siphon
    E["📡<br/>hass<br/>sink"]:::siphon
    F["🏠<br/>Home Assistant"]:::ha

    A -->|"HTTPS<br/>POST"| B
    B -->|"tcp<br/>proxy"| C
    C -->|"internal<br/>event bus"| D
    D --> E
    E -->|"MQTT<br/>publishes PRESS"| F
```

### Step 1: Expose Siphon via Tailscale Funnel

[Tailscale](https://tailscale.com) is a zero-config VPN built on WireGuard. **Funnel** is its feature that exposes a
local service to the public internet via a stable `*.ts.net` hostname with automatic Let's Encrypt TLS — no port
forwarding or DNS required.

> Tailscale is purely my personal choice — other solutions exist. That said, for home lab and hobbyist use, Tailscale
> hits a sweet spot that's hard to beat: the free tier covers personal use generously, setup takes minutes rather than
> days, and you get a properly signed TLS certificate and a stable public hostname without touching your router or
> registering a domain. It removes the entire class of "expose this to the internet" problems that usually involve
> either a cloud VPS, a DDNS hack, or opening holes in a firewall — none of which are great when you just want a Garmin
> watch to poke a button. It's also genuinely portable: clients exist for Android, iOS, Windows, Linux, and — critically
> for this setup — there are packages for OpenWrt, so it runs directly on the router.

Tailscale's funnel feature provides a public HTTPS endpoint that terminates TLS and proxies traffic to a local service.
Our setup has a twist: `tailscale` daemon runs on the OpenWrt router, while **Siphon** runs on the Home Assistant VM.
Since `tailscale funnel` only accepts `localhost` as an upstream (this statement is valid for version `1.80.3` of
`tailscale` daemon which is the newest available on my router currently, _newer versions should not require the `socat`
workaround anymore_), we bridge the gap with `socat`:

```bash
# On the OpenWrt router
opkg update && opkg install socat

# Forward localhost:8888 → HA VM:8080 (bind to loopback only)
socat TCP-LISTEN:8888,bind=127.0.0.1,fork,reuseaddr TCP:192.168.6.5:8080 &

# Expose via Funnel
tailscale funnel --bg http://localhost:8888
```

> To survive reboots, I've deployed a `procd` init script at `/etc/init.d/siphon-funnel` with `respawn` — procd will
> restart socat automatically if it crashes.

The result is a stable public endpoint: `https://{node}.{tailnet name}.ts.net`.

### Step 2: The Siphon Pipeline

The pipeline is three blocks in `config.yaml`:

**Collector** — listens for incoming POST requests, authenticates via `Bearer` token, deduplicates by payload hash
within a 1-second window to prevent double-triggers, if hands are shaky:

```yaml
collectors:
  garmin_wh:
    type: webhook
    topics:
      garmin_raw: /webhook/garmin
    params:
      port: 8080
      token: "%%WEBHOOK_TOKEN%%"
      dedupe_ttl: 1
```

**Sinks** — two complementary sinks handle the event: `garmin_trigger` registers a `device_automation` trigger in **Home
Assistant** via **MQTT Auto-Discovery** (requires v0.5.4+) — no entity required, pure event pushing. `garmin_trigger_cnt` is a
`total_increasing` sensor that tracks the cumulative press count — for statistical reasons, not exactly needed:

```yaml
sinks:
  garmin_trigger:
    type: hass
    params:
      url: "%%MQTT_HOST%%"
      user: "%%MQTT_USER%%"
      pass: "%%MQTT_PASS%%"
      object_id: garmin_button
      component: device_automation
      trigger_type: button_short_press
      subtype: button_1
  garmin_trigger_cnt:
    type: hass
    params:
      url: "%%MQTT_HOST%%"
      user: "%%MQTT_USER%%"
      pass: "%%MQTT_PASS%%"
      object_id: garmin_count
      component: sensor
      state_class: total_increasing
      icon: "mdi:watch"
      value_template: "{{ value_json.count }}"
```

**Pipeline** — `stateful`: a running `cnt` is maintained across events. Each webhook hit publishes a discrete `PRESS` to
the device trigger action topic; the counter sink records the total. The webhook body is ignored — no `parser` block
is needed, any incoming POST triggers the pipeline:

```yaml
pipelines:
  - name: garmin
    stateful: true
    topics:
      - garmin_raw
    transform:
      - cnt: (cnt ?? 0) + 1
    sinks:
      - name: garmin_trigger
        format: expr
        spec: '"PRESS"'
      - name: garmin_trigger_cnt
        format: expr
        spec: |
          { "count" : cnt ?? 0 }
```

### Step 3: The Home Assistant Automation

With the device trigger registered, wire up an automation. Siphon registers itself as a device under the MQTT
integration — use the `device_id` shown in HA's device registry:

```yaml
alias: Garmin trigger
description: Fires on each Garmin {API}Call press via Siphon webhook
triggers:
  - domain: mqtt
    device_id: {Siphon's device ID}
    type: button_short_press
    subtype: button_1
    trigger: device
conditions: []
actions:
  - action: light.toggle
    metadata: {}
    target:
      entity_id: light.living_room
    data: {}
mode: single
```

The trigger fires on the `button_short_press` / `button_1` combination declared in the sink. No polling, no state entity
— the press lands in HA as a native device event and just toggles the light in the living room.

![The device in Home Assistant](images/20260421/hass_device.webp)

### Step 4: The {API}Call configuration

This configuration was used (the `\\\"` escaping is required by the Connect IQ config UI parser):

```json
{deviceName:"Home",actionName:"Siphon",url:"https://{node name}.{tailnet name}.ts.net/webhook/garmin", headers:"{Authorization:\\\"Bearer {token}\\\"}", method: "POST", POSTcontent:"{action: \\\"triggered\\\"}"}
```

The `POSTcontent` value is essentially ignored here, but it can be used to verify its content in the pipeline if needed.
More important is the **Bearer token**, which only allows the request through when the device is authorized.

![Garmin watch, {API}call triggered](images/20260421/garmin.webp)

## v0.5.x Patch notes

### v0.5.4 (2026-04-20)

- **hass sink: native MQTT Device Triggers** (`component: device_automation`) for stateless event pushing — no entity
  lifecycle required.
- hass sink: unified, data-driven topic system — a `componentTopics` map drives both discovery payload generation and
  `Send()` routing for all supported HA component types.
- hass sink: consistent auto-generation and per-topic overrides for `state_topic`, `command_topic`, and
  `availability_topic`.

### v0.5.3 (2026-04-19)

- fix webhook collector: duplicate payloads now correctly return `409 Conflict` instead of `200 OK`.

### v0.5.2 (2026-04-19)

- fix webhook collector: `dedupe_ttl` was not respected — cleanup ticker was hardcoded to 1 minute regardless of
  configured TTL value.

### v0.5.1 (2026-04-19)

- General refactor and added unit tests.

### v0.5.0 (2026-04-13)

- Add editor status line displaying current version.
- Add hot reload support.
- Transform block changed to array for deterministic ordering.
- Added topic as selector for variables.
- Added `rest` and `webhook` collectors.
- Added `mqtt` and `hass` sinks.
- Sinks: added `Close()` to interface for clean shutdown of long-running sinks (e.g. MQTT).
