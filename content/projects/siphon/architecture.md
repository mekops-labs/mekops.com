---
title: "Core Architecture"
date: 2026-04-06T12:05:00+02:00
weight: 10
description: "Understanding the V2 decoupled Event Bus and data lifecycle."
---

Siphon v0.5 utilizes a decoupled, stateful Event Bus architecture. The data flow operates in a strict, linear pipeline designed to isolate data ingestion from state management and routing.

### The Data Lifecycle

1. **Collectors (The Ingress):** Actively subscribe to or poll configured sources. Their only job is to acquire raw bytes and push them onto the internal Event Bus with a logical alias. Supported engines: `mqtt`, `rest`, `webhook`, `file`, `shell`, and the native `hass` Supervisor API.
2. **Parsers (The Sandbox):** Pipelines subscribe to aliases on the Event Bus. They use integrated parser engines (`jsonpath`, `regex`) to extract structured data from raw payloads and safely cache the exact state in memory.
3. **Dispatchers (The Schedulers):** Determine *when* data should be pushed.
   * **Cron:** Uses standard cron syntax to wake up, pull the latest cached states from multiple pipelines, and dispatch them simultaneously.
   * **Event:** Triggers immediately upon a value change or threshold breach.
4. **Sinks (The Egress):** Format the state using the embedded `expr` evaluation engine and transmit to final destinations. Supported sinks: `hass` (MQTT Auto-Discovery — stateful entities via `sensor`, `binary_sensor` and stateless event triggers via `device_automation`), `mqtt`, `gotify`, `ntfy`, `windy`, `iotplotter`, `stdout`.

### Zone Isolation Model

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': { 'primaryColor': '#1a1a1a', 'edgeLabelBackground':'#1a1a1a', 'tertiaryColor': '#94a3b8', 'lineColor': '#94a3b8', 'textColor': '#94a3b8'}}}%%
graph LR
    classDef public fill:#2a1215,stroke:#7f1d1d,stroke-width:1px,color:#ef4444;
    classDef collector fill:#0f172a,stroke:#2563eb,stroke-width:1px,color:#60a5fa;
    classDef processor fill:#281809,stroke:#b45309,stroke-width:1px,color:#fbbf24;
    classDef sink fill:#064e3b,stroke:#059669,stroke-width:1px,color:#34d399;
    classDef internal fill:#1a1a1a,stroke:#334155,stroke-dasharray: 5 5,color:#94a3b8;
    classDef secure fill:#1a1a1a,stroke:#059669,stroke-width:1px,color:#94a3b8;

    subgraph Zone1 ["🌐 INTERNET"]
        direction LR
        PublicNet[["untrusted<br/>webhooks"]]:::public
    end

    subgraph Zone2 ["⚙️ DMZ (SIPHON DAEMON)"]
        direction LR
        Collect1("🏰<br/>webhook<br/>collector"):::collector
        BusVolatile[("⚡<br/>event<br/>bus")]:::internal
        Proc("🔪<br/>processors<br/>('expr' extract<br/>and filter)"):::processor
        Sink1("🌉<br/>MQTT<br/>sink"):::sink

        Collect1 --> BusVolatile
        BusVolatile --> Proc
        Proc -->|valid<br/>payload| Sink1
    end

    subgraph Zone3 ["🔒 SECURE LOCAL NETWORK (No Ingress)"]
        direction LR
        Broker[("📡<br/>local MQTT<br/>broker")]:::secure -->|Auto-Discovery| HA[("🏠<br/>Home<br/>Assistant")]:::secure
    end

    PublicNet -->|https| Collect1
    Sink1 -->|MQTT<br/>over TLS| Broker
```
