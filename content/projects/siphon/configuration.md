---
title: "Configuration Reference"
date: 2026-04-06T12:15:00+02:00
weight: 30
toc: true
description: "Writing V2 YAML schemas to map collectors, pipelines, and sinks."
---

Siphon uses a strict, declarative YAML configuration.

Example config: {{< github "mekops-labs/siphon/blob/main/configs/example.yaml" >}}

---

## Env variables substitution

Whole configuration file is first parsed and all `%%ENV_NAME%%` references are
substituted by values of environmental variables seen by siphon.

---

## The Global Block

The root of the document requires the V2 schema declaration.

```yaml
version: 2
```

---

## Collectors

Collectors define how Siphon ingests data. Each collector is given a unique alias and maps raw input parameters to
internal Event Bus topics.

### `mqtt`

{{< github "mekops-labs/siphon/blob/main/pkg/collector/mqtt/README.md" >}}

Used to subscribe to one or more topics on a remote or local MQTT broker.

Parameters (`params`):

- `url` (required): The connection string (e.g., tcp://127.0.0.1:1883 or ssl://broker.example.com:8883).
- `user`: Username for authentication.
- `pass`: Password for authentication.

Topics (`topics`): A map where the key is the internal Siphon bus topic (alias) and the value is the actual MQTT topic string.

### `file`

{{< github "mekops-labs/siphon/blob/main/pkg/collector/file/README.md" >}}

Reads the content of local system files, typically used for monitoring hardware sensors or system metrics.

Parameters (`params`):

- `interval`: Frequency of polling in seconds (default: 10).

Topics (`topics`): A map where the key is the internal bus topic and the value is the absolute path to the file.

### `shell`

{{< github "mekops-labs/siphon/blob/main/pkg/collector/shell/README.md" >}}

Executes an OS command periodically and captures its stdout.

Parameters (`params`):

- `interval`: Frequency of execution in seconds (default: 10).

Topics (`topics`): A map where the key is the internal bus topic and the value is the shell command to run.

### `hass`

{{< github "mekops-labs/siphon/blob/main/pkg/collector/hass/README.md" >}}

A specialized collector for Home Assistant. It polls entity states via the REST API. When running as an official Add-on,
it automatically handles authentication using the `$SUPERVISOR_TOKEN`.

Parameters (params):

- `interval`: Polling frequency in seconds (default: 30)

Topics (`topics`): A map where the key is the internal bus topic and the value is the Home Assistant entity ID (e.g.,
`sensor.living_room_temp`). Supports wildcard `*` value - in this case returns all the entities in JSON format.

### Example collector definition

```yaml
collectors:
  local_ha:
    type: hass
    params:
      interval: 60
    topics:
      outdoor_temp: "sensor.backyard_temperature"
```

---

## Sinks

Sinks define the egress points for your telemetry data. They are defined globally in the `sinks` block and can be
referenced by name within multiple pipelines.

### `bus`

{{< github "mekops-labs/siphon/blob/main/pkg/sink/bus/README.md" >}}

Publishes the processed payloads back to the internal bus.

Parameters (`params`):

- `topic` (required): The internal bus topic to publish to.

### `gotify`

{{< github "mekops-labs/siphon/blob/main/pkg/sink/gotify/README.md" >}}

Sends push notifications to a Gotify server.

Parameters (`params`):

- `url` (required): The base URL of your Gotify instance.
- `token` (required): The application token.
- `title`: The notification title. This fields uses template rendering. `now` function is defined as alias to
  time.Now().Format() (default "Siphon Alert")
- `priority`: The notification priority (default 0)

### `ntfy`

{{< github "mekops-labs/siphon/blob/main/pkg/sink/ntfy/README.md" >}}

Sends notifications to ntfy.sh or a self-hosted instance.

Parameters (`params`):

- `url` (required): The ntfy instance URL (e.g., `https://ntfy.sh`).
- `topic` (required): The ntfy topic
- `token`: The ntfy token (default: "")
- `title`: The notification title. This fields uses template rendering. `now` function is defined as alias to
  time.Now().Format() (default: "")
- `priority`: The notification priority (default: 3)

### `windy`

{{< github "mekops-labs/siphon/blob/main/pkg/sink/windy/README.md" >}}

Updates a Personal Weather Station (PWS) on Windy.com.

Parameters (`params`):

- `id` (required): Your Station ID.
- `password` (required): Your Windy PWS password from station's page.

### `iotplotter`

{{< github "mekops-labs/siphon/blob/main/pkg/sink/iotplotter/README.md" >}}

Dispatches data to the IoTPlotter service.

Parameters (`params`):

- `url`: The API endpoint (default: `http://iotplotter.com`).
- `apikey` (required): Your account API key.
- `feed` (required): The specific feed ID to update.

### `stdout`

{{< github "mekops-labs/siphon/blob/main/pkg/sink/stdout/README.md" >}}

Prints the final payload to the Siphon logs. Useful for debugging.

Parameters (`params`):

- None.

### Example sink definition

```yaml
sinks:
  local_debug:
    type: stdout
  mobile_alerts:
    type: gotify
    params:
      url: "https://gotify.example.com"
      token: "%%GOTIFY_TOKEN%%"
```

---

## Pipelines & Parsers

Pipelines are the primary processing units in Siphon. They act as consumers of the internal Event Bus, defining how
raw data is transformed and where it is eventually dispatched.

### Pipeline Fields

- `name` (required): A unique identifier for the pipeline.
- `type`: Defines the trigger mechanism.
  - `event` (default): Executes as soon as new data arrives on a subscribed topic.
  - `cron`: Executes based on a temporal schedule.
- `topics` (required): A list of internal bus topic aliases this pipeline subscribes to.
- `schedule`: Required if `type` is `cron`. Uses standard cron syntax (e.g., `*/5 * * * * *`).
- `stateful`: (boolean) When `true`, Siphon caches the last known variables from this pipeline's parser. This is
  essential for merging data from multiple sources in a `cron` pipeline.
- `bus_mode`:
  - `volatile` (default): High-performance in-memory processing.
  - `durable`: (WIP) Persists events to a Write-Ahead Log to prevent data loss during restarts.

### The `parser` Block

The parser converts raw bytes from the collector into structured variables.

- `type`: The engine used for extraction.
  - `jsonpath`: For structured JSON payloads.
  - `regex`: For extracting values from unstructured strings or logs.
- `vars`: A map where the key is the variable name and the value is the path or pattern.

### The `transform` Block (Optional)

An optional map of `expr` expressions to further process variables before they hit the sinks.

```yaml
transform:
  temp_f: "(temp_c * 9/5) + 32"
```

### The `sinks` Array

Defines the egress routing for the pipeline.

- `name`: The alias of a globally defined sink.
- `format`: The rendering engine used for the payload.
  - `expr`: Uses the expression engine to build complex JSON or logic.
  - `template`: Uses standard Go `text/template` syntax.
- `spec`: The actual expression or template string defining the output.

---

### Example pipeline definition

```yaml
pipelines:
  - name: process_garage
    type: event # Triggers instantly when new data arrives
    topics: ["garage_telemetry"]
    parser:
      type: jsonpath
      vars:
        door_state: "$.door.open"
        battery_v: "$.voltage"
    transform:
      door_state: "string(door_state)"
      battery_v: "battery_v / 1000"
    sinks:
      - name: alert_phone
        format: expr
        spec: |
          {
            "title": "Garage Update",
            "door": door_state,
            "battery": battery_v
          }
```

### Advanced: Stateful Cron Merging

Because the Event Bus caches the parsed state, a cron pipeline can wake up on a schedule, fetch the latest states from
completely different collectors, and merge them into a single payload using the expr engine.

```yaml
pipelines:
  - name: outdoor_temp
    topics: ["outdoor"]
    parse:
      type: jsonpath
      vars:
        state: "$.state"

  - name: garage_telemetry
    topics: ["garage"]
    parse:
      type: jsonpath
      vars:
        battery_v: "$.voltage"

  - name: unified_dispatcher
    type: cron
    stateful: true
    schedule: "*/30 * * * * *"  # Run every 30 seconds
    topics: ["outdoor_temp", "garage_telemetry"]
    sinks:
      - name: my_backend
        type: stdout
        format: expr
        spec: |
          {
            "merged_telemetry": {
              "temp": state ?? 0,
              "garage_v": battery_v ?? 0
            }
          }
```

> Note the use of the `??` null-coalescing operators in the expr block to prevent panics if a collector hasn't reported
> data yet.
