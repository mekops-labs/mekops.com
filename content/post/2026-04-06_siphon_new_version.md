---
title: "Project Siphon: v0.4 Architecture, The HA Add-on, and The Web UI"
author: Kamil Wcisło
date: 2026-04-06
draft: false
toc: true
series: siphon
tags:
  - home_assistant
  - etl
  - iot
thumbnail: images/20260406/hass.webp
summary: |
    With the release of the v0.4.x series, Siphon has officially outgrown its initial "script replacement" phase. I have
    completely stripped and rebuilt the core memory bus, introduced a highly opinionated v2 declarative schema, and,
    most importantly, packaged the entire engine as a native Home Assistant Add-on.
---

> [Siphon's project page](/projects/siphon).

Building telemetry pipelines is inherently messy. Every new sensor, webhook, or external API wants to dictate how you
receive data. For the last few months, Siphon existed to solve that problem by acting as a rigid, decoupled gateway
between the chaotic outside world and your pristine internal data bus, but it evolved.

Here is a breakdown of the architectural shifts and how you can deploy.

## The V2 Engine: Decoupled by Design

The biggest flaw in the v1 architecture was coupling. A pipeline cared too much about how a collector acquired data. The
v2 engine fixes this by introducing an isolated, asynchronous Event Bus.

"Dumb" Collectors Ingestion engines (mqtt, file, webhook, shell) are now purposefully ignorant of the data they consume.
Their only job is to acquire raw bytes, attach a logical alias to them, and throw them onto the bus.

### Stateful Cron Merging

Because the Event Bus caches parsed state in memory, we can now do things that previously required
complex Python scripts. You can configure a cron pipeline to wake up on a schedule, grab the latest cached states from
completely different collectors, and merge them into a single, sanitized JSON payload before firing it off to a sink.

### The Pipeline Sandbox

Here is what merging local system thermals via file and room sensors via mqtt into a unified payload looks like in the
new schema:

```yaml
version: 2

collectors:
  mqtt_broker:
    type: mqtt
    params: { url: "tcp://192.168.1.1:1883" }
    topics:
      sensor_raw: "sensor/room" # Alias -> MQTT Topic

  sys_metrics:
    type: file
    params: { interval: 10 }
    topics:
      cpu_raw: "/sys/class/thermal/thermal_zone0/temp"

pipelines:
  # Ingest and Parse MQTT Data
  - name: mqtt_state
    topics: ["sensor_raw"]
    parser:
      type: jsonpath
      vars: { temp: "$.T" }

  # Ingest and Parse File Data
  - name: file_state
    topics: ["cpu_raw"]
    parser:
      type: regex
      vars: { temp_str: "[0-9]+" }

  # Merge them using the internal Expr engine
  - name: unified_dispatcher
    type: cron
    stateful: true
    schedule: "*/30 * * * * *" # every 30s
    topics: ["mqtt_state", "file_state"]
    sinks:
      - name: my_backend
        format: expr
        spec: |
          {
            "merged_data": {
              "room_temperature": temp ?? 0,
              "server_cpu": temp_str ?? 0
              }
          }
```

## Siphon in Home Assistant

While you can still deploy Siphon in an isolated Docker container on your DMZ (the purist approach), I wanted an escape
hatch. Siphon is now a first-class citizen in Home Assistant, installable directly via a custom [Add-on
repository](https://github.com/mekops-labs/siphon-ha-addon).

### Zero-Setup Auth & Environment Isolation

If Siphon detects the Mosquitto broker Add-on, it automatically injects the credentials into the environment. You use
`%%MQTT_HOST%%`, `%%MQTT_PORT%%`, `%%MQTT_USER%%`, and `%%MQTT_PASS%%` in your config without ever hardcoding a
password. Furthermore, you can define external environment variables in your config directly in the Add-on UI, keeping
your config.yaml clean and safe to commit to version control. The defined variables could be injected to config using
`%%ENV_VARIABLE%%` syntax.

### The Native hass Collector

Because running as an Add-on grants Siphon a `$SUPERVISOR_TOKEN`, I wrote a new native `hass` collector. It leverages
this token to poll Home Assistant entities directly via the internal REST API. No Long-Lived Access Tokens, no external
network calls. Just clean, local, secure polling.

```yaml
collectors:
  local_ha:
    type: hass
    params:
      interval: 60
    topics:
      outdoor_temp: "sensor.backyard_temperature"

pipelines:
  # Process the data from Home Assistant
  - name: process_ha_temp
    topics: ["outdoor_temp"] # Listen to the alias!
    bus_mode: volatile
    parser:
      type: jsonpath
      vars:
        # The HA API always returns a standard JSON object.
        # You can extract the main state or nested attributes.
        temp: "$.state"
        unit: "$.attributes.unit_of_measurement"

    # Transform...
    transform:
      tempStr: "string(float(temp))+' '+unit"
```

> Some more examples are described on
[github](https://github.com/mekops-labs/siphon/blob/main/pkg/collector/hass/README.md).

## The Web Interface

SSH-ing into a container to edit a YAML file gets old fast. Siphon now features an embedded web UI powered by
**Ace.js**, accessible directly through Home Assistant Ingress.

It provides a fast, syntax-highlighted editor directly in your HA sidebar.

## What's Next

The v0.4 release also includes a complete rewrite of the Windy PWS sink to support their new 2026 API requirements,
alongside fixes for the gotify and ntfy integrations.

Siphon is officially ready to handle your edge telemetry. Check out the GitHub repository to grab the Add-on URL or pull
the latest Docker image. Let Home Assistant run the automations; let Siphon guard the gate.

{{< github "mekops-labs/siphon" >}} {{< github "mekops-labs/siphon-ha-addon" >}}
