---
title: "WANTED is Public: The Repo, the Docs, and What's Actually Built"
author: Kamil Wcisło
date: 2026-06-16
toc: true
series: wanted
thumbnail: images/20260616/wanted-public.webp
tags:
  - edge-computing
  - embedded
  - iot
  - webassembly
summary: "I promised an alternative to the firmware monolith. The WANTED engine is public now — here's what actually builds and runs, and what's still a TBD."
---

Last post I laid out the uncomfortable truth: you wouldn't deploy your backend as a single statically-linked binary on
an unpatched server — yet that's exactly how we treat the µCs running our homes and factories. I argued that the
monolith brings fragility, operational friction, and a codebase nobody wants to touch.

I said there was an alternative, and that it would be public soon.

It is public now.

## The repository and the docs

I'd rather show you what's solid, what's half-built, and what's still a sketch than dress a side-project runtime up as a
finished product. So here's the whole thing, warts and all — engine source, the test suite, and the sample Wapp builds:
- {{< github "mekops-labs/wanted-engine" >}}

It's **Apache 2.0** — fork it, ship it, vendor it into something bigger; no license friction with the rest of the
cloud-native stack.

For a runtime nobody's seen yet, the docs matter as much as the code, so they went up alongside it — architecture, the
runtime spec, the Sheriff control plane, and the capability model:
- **[Project page](projects/wanted)**

That landing page is the canonical reference for the rest of this series.

## The stack

WANTED is more than the runtime; it's a full stack for edge autonomy, in three layers:

1.  **Runtime:** A WebAssembly Micro Runtime (WAMR)-based engine that provides a sandboxed, capability-gated environment
    for application code.
2.  **Supervisor (Sheriff):** An on-device control-plane agent that manages the Wapp lifecycle, enforces resource
    limits, and reconciles desired state against the physical reality of the device (will also be public soon).
3.  **Cloud (Marshal/Deputy):** The control-plane layer, currently in design. Marshal targets Kubernetes-native GitOps
    orchestration; Deputy is a lightweight standalone binary for local orchestration with no cluster dependencies — a
    single binary with a web UI and optional MQTT Discovery for Home Assistant users.

## The honesty table

This is a one-person project in active early development, so here's what actually compiles and passes CI today versus
what's still a branch or a sketch — no roadmap theater:

| State | Components |
|-------|-----------|
| **Built & merged** | VFS router (namespace isolation), TarFS + OCI layered images, WAMR 2.4.4 classic-interpreted runtime, Linux platform, NuttX simulation target (CI-gated, functional selftest suite), named-pipe IPC, per-wapp console driver, WASI `args`/`environ` passthrough, the on-device Sheriff supervisor (initial control-plane primitives) - currently in compiled wasm form in repo, but source code will be available. |
| **In progress** | NuttX ESP32 hardware, the Sheriff reconciliation loop, Deputy and Marshal control planes (in design). |
| **Research** | Checkpoint/Restore, delta OTA, Execute-In-Place (XIP) from Flash, RISC-V target support. |

## Try it today

No exotic hardware required: clone it, build on Linux, and have a "hello from the edge" Wapp running in a few minutes.
The [Quickstart](projects/wanted/quickstart) walks the whole loop — compile to `wasm32-wasi`, package it into an OCI
image, and launch it through the control plane.

## What's coming

This was orientation. The next posts go subsystem by subsystem, bytecode up:

*   **Part 3:** WebAssembly on Bare Metal
*   **Part 4:** Hardware as Files: The VFS Capability Model
*   **Part 5:** OCI Images on an RTOS
*   **Part 6:** The Control Plane
*   **Part 7:** Real Deployments
*   **Part 8:** Checkpoint/Restore

Star the repo if it's useful, follow the series, and open an issue when I get something wrong — this is the edge
infrastructure I wanted and couldn't buy, built in the open.
