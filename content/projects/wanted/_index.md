---
title: "WANTED Engine"
date: 2026-06-14T20:00:00+01:00
draft: false
description: "A WebAssembly nanocontainer runtime that isolates and runs multiple apps as threads, with all I/O mediated through a cloud-native VFS router."
thumbnail: images/wanted/logo.svg
layout: "single"
summary: |
  WANTED runs multiple WebAssembly applications — **wapps** — as isolated threads inside one process. A wapp's only
  interface to the outside world is a virtual filesystem: every device, socket, IPC channel, and control-plane node
  is a path. This page is the conceptual anchor; the reference pages drill into each surface.
---

![::project-logo](images/wanted/logo.svg)
{{< gitlab "mekops/wanted/wanted-engine" >}} {{< github "mekops-labs/wanted-engine" >}}
![::inline-badge](https://img.shields.io/github/v/tag/mekops-labs/wanted-engine?style=flat&label=ver)
![::inline-badge](https://img.shields.io/github/license/mekops-labs/wanted-engine)

WANTED is **W**eb**A**ssembly **N**anocontainer **T**echnology for **E**mbedded **D**evices. It loads, isolates, and runs multiple WebAssembly applications — **wapps** — as independent threads inside a single process. Each wapp runs in its own WASM linear memory and reaches the outside world only through a virtual filesystem: hardware, network sockets, IPC channels, and the runtime control plane are all paths in a per-wapp namespace. There is no ambient host access and no shared memory between wapps.

The engine targets constrained hardware first. It runs on Linux today and on NuttX (host simulator), uses the WAMR interpreter so no per-target codegen is needed, and packages wapps as OCI-compatible layered TAR images for delta updates and offline distribution.

## Key properties

- **Offline-first.** Wapps are self-contained OCI TAR images; no registry connectivity is required to install or run them.
- **Capability isolation via the VFS.** A wapp can only touch what its launch config mounts into its namespace — `/dev/`, `/net/`, `/proc/`, `/etc`. No grant, no access.
- **Standard WASI ABI.** Wapps are ordinary `wasm32-wasi` binaries; the host interface is WASI `snapshot_preview1` plus a small VFS-mediated control surface.
- **OCI-compatible packaging.** Layered ustar TARs with shadowing and whiteout semantics, indexed for O(log N) lookup and zero-copy boot.
- **Portable across targets.** A thin `Platform*` seam abstracts threads, sockets, files, clock, and memory stats; Linux and NuttX are the two production implementations.

## Feature matrix

| Area          | Capability |
|---------------|------------|
| Runtime       | WAMR 2.4.4, fast interpreter; thread-per-wapp execution; WASI `snapshot_preview1` bridge      |
| VFS drivers   | TarFS root, DevFS (`/dev/`), NetFS (`/net/`), ProcFS (`/proc/`), named pipes, sockets (TCP/UDP/TLS), 9P client, log console |
| Packaging     | OCI-compatible layered ustar TAR; up to 4 layers; whiteout deletion; PAX/GNU long names       |
| Supervisor    | Privileged wapp loaded at boot; variants for production control (sheriff), interactive debug (wsh), and in-WASM self-test |
| Control plane | `/dev/wanted/*` — install, start/stop, observe state, read logs, and drive engine power state |
| Platforms     | Linux (primary); NuttX simulator (CI-gated); NuttX on ESP32 hardware coming soon              |

## Documentation

{{< children >}}

## Source

The engine is developed in the open at [gitlab.com/mekops/wanted/wanted-engine](https://gitlab.com/mekops/wanted/wanted-engine), with mirror on [github.com/mekops-lab/wanted-engine](github.com/mekops-lab/wanted-engine)

Contributions welcome! You can open issue on github and/or gitlab (where CI lives). MR/PR are accepted on both places.
