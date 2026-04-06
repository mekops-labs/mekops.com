---
title: "Siphon"
date: 2026-03-30T18:00:00+02:00
draft: false
thumbnail: projects/siphon/siphon.webp
description: "A zero-trust edge gateway and telemetry pipeline."
layout: "single"
summary: |
  Siphon is a modular IoT ETL (Extract, Transform, Load) engine in Go.
  It ingests data (MQTT, File, Shell), normalizes it via dynamic parsers
  (JSONPath, Regex), and dispatches it to sinks (REST, MQTT).
---

![::project-logo](projects/siphon/siphon.webp)
{{< github "mekops-labs/siphon" >}} {{< github "mekops-labs/siphon-ha-addon" >}}
![::inline-badge](https://img.shields.io/github/v/tag/mekops-labs/siphon?style=flat&label=ver)
![::inline-badge](https://img.shields.io/github/license/mekops-labs/siphon)

**Siphon** is a lightweight, dynamically configurable data aggregation engine written in Go. Designed for edge computing, containerized deployments, and native Home Assistant integration, Siphon acts as a secure, zero-trust middleware layer.

It gathers raw telemetry data from untrusted public webhooks, local system files, or MQTT brokers, ruthlessly normalizes it using an integrated expression engine, and securely dispatches it to internal smart home networks or external REST APIs.

## Documentation

Explore the official guides to understand the V2 event bus, deploy the daemon, and configure your pipelines:

{{< children >}}

## Roadmap

The project is still WIP. My current direction is to make it a viable Home Assistant extension with support for MQTT
auto discovery, then I plan to polish the project.

### Phase 1: Core Engine Refactor (V2 Architecture)

**Focus: Implementing the Hybrid Event Bus and the new linear pipeline configuration.**

- ✅ Config V2 Schema: Define config.go structs to parse the new pipelines array and generic parameters.
- ✅ Hybrid Bus Engine: Build HybridBus supporting both ModeVolatile (ring buffers) and ModeDurable (WAL).
  - ☐ ModeDurable is still TODO
- ✅ Modules Refactor: Update all existing Collectors to publish to the HybridBus and handle backend delivery failures.
- ✅ Sink Refactor: Update all existing Sinks to subscribe to the HybridBus and explicitly call event.Ack() upon success.

### Phase 2: Home Assistant Native Ecosystem

**Focus: Seamless integration, auto-discovery, and providing a premium Add-on experience.**

- ☐ HA Config Structs: Define DiscoveryConfig and map YAML fields for HA metadata (Device Class, Node ID, etc.).
- ☐ HA MQTT Payloads: Implement JSON-tagged DiscoveryPayload and DevicePayload structs.
- ☐ MQTT Reliability: Implement Last Will and Testament (LWT) logic in the MQTT Sink.
- ☐ Auto-Discovery Hooks: Create Announce() method to publish retained discovery configs upon MQTT connection.
- ✅ Add-on Repository: Create the standalone HA Add-on structure.
- ✅ Ingress Web Server: Implement an embedded Go web server in pkg/editor.
- ✅ Embedded UI: Create an index.html using Ace.js for live config.yaml editing via HA Ingress.
- ✅ Hass Collector: Directly attach to Home Assistant API to get entity states.

### Phase 3: API Gateway & Synchronous Processing

**Focus: Transforming Siphon into a two-way sync engine for devices like wearables.**

- ☐ Reply Channels: Add a ReplyTo channel to the Event struct to support synchronous routing.
- ☐ Webserver Collector: Implement a Collector that keeps HTTP requests open while waiting for the pipeline to finish.
- ☐ Responder Sink: Implement a Sink that formats pipeline output and writes it back to the ReplyTo channel.

### Phase 4: Benchmarking & Performance

**Focus: Proving reliability and measuring tail latency under extreme load.**

- ☐ Harness Setup: Create an E2E testing harness (cmd/bench/main.go) utilizing a local MQTT broker.
- ☐ Precision Metrics: Integrate `hdrhistogram-go` for microsecond-precision latency and jitter tracking.

### Phase 5: Advanced Extensibility & Polish

**Focus: Documentation, tooling, and future-proofing the parser engine.**

- ☐ JSON Schema: Generate a complete JSON Schema for config.yaml (v2).
- ☐ Wasm: WebAssembly processors support (plugin system), TBD.
