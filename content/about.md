+++
title = "About"
description = "About me"
date = "2026-01-24"
aliases = ["about-me", "contact"]
author = "Kamil Wcisło"
summary = "The philosophy behind MekOps and the engineer building the stack."
comments = false
+++

# The Bridge Between Cloud and Metal

Hello. I’m **Kamil Wcisło**, a Systems Engineer specializing in **Cloud-Native Embedded Systems** and my project,
**MekOps** stands for **M**icroservices, **E**mbedded, and **K**ernels **Op**eration**s**.

In the modern tech landscape, there is a significant divide. _Cloud_ engineers treat infrastructure as infinite and
disposable. _Embedded_ engineers treat hardware as static and specific. I would like to introduce an engineering
philosophy that applies the scalability, observability, and orchestration of the _Cloud_ to the rigid, constrained and
multi-architectural reality of _bare metal_.

I build tools that allow physical hardware to be managed with the same agility as a Kubernetes cluster, without
sacrificing the performance of C and Rust. To build the next generation of edge computing, you cannot just be an
embedded engineer, and you cannot just be a cloud architect. You must own the stack from the **Kernel**
(NuttX/Linux/Rust/C) to the **Microservice** (Go/Wasm) and finally **Operations** (Kubernetes).

### What I Build

I don't just use tools; I forge them. My work focuses on the intersection of **Kubernetes**, **WebAssembly**, and
**Real-Time Operating Systems**.

<!-- Currently, I am architecting the **MekOps Ecosystem**, a vertically integrated stack for the edge:

* **[WANTED](https://github.com/mekops-labs/wanted-runtime):** A container runtime for microcontrollers. It uses
  WebAssembly to bring "Docker-like" isolation to devices that are too small for Linux.
* **[MARSHAL](https://github.com/mekops-labs/marshal):** A Kubernetes Operator that extends the control plane to
  physical devices, treating their fleet as just another set of nodes.
* **[SIPHON](https://github.com/mekops-labs/siphon):** A high-performance telemetry pipeline written in Go, designed to
  extract signal from noise in constrained bandwidth environments. -->

### My Philosophy

#### 1. Privacy is an Architecture, Not a Setting.
I believe that data should be owned by the creator, not the platform. My tools are designed to be self-hosted,
air-gapped, and audit-friendly. I reject the "cloud-first" default that demands constant connectivity.

#### 2. Complexity Should Be Optional.
Modern DevOps is drowning in abstraction. I prefer **"Boring Technology"**—simple binaries, typed interfaces, and
standard protocols (MQTT, HTTP, POSIX). If it requires a SaaS subscription to run, I won't use it.

#### 3. The Hardware Matters.
Software optimization is a lost art. I write code that respects the silicon it runs on. Whether it's porting **Apache
NuttX** to a new LoRa pager or shaving bytes off a Wasm binary, I care about the cycles.

---
