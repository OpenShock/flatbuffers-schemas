# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Shared [FlatBuffers](https://google.github.io/flatbuffers/) schema definitions for the OpenShock ecosystem. Used as a git submodule by the firmware (ESP32 C++), API/LCG (.NET), and other components. Also published as the `OpenShock.Serialization.Flatbuffers` NuGet package via FlatSharp.

## Build

```bash
dotnet build   # FlatSharp compiler runs automatically, generating C# bindings from .fbs files
```

Targets: net10.0, net9.0, net8.0, netstandard2.1. FlatSharp v7.9.0.

## Schema Conformity Verification

CI runs `flatc --conform` against the master branch for schemas listed in `.github/workflows/verify-list`. This catches backward-incompatible changes. To test locally:

```bash
flatc new/HubConfig.fbs --conform master/HubConfig.fbs
```

## Protocol Architecture

```
Gateway (LCG) ──FlatBuffers WebSocket──> Hub (Firmware)
  GatewayToHubMessage.fbs                  HubToGatewayMessage.fbs

Hub (Firmware) ──FlatBuffers──> Local Web UI (Captive Portal)
  HubToLocalMessage.fbs                    LocalToHubMessage.fbs
```

- **Gateway <-> Hub**: Shocker commands, OTA updates, keep-alive pings, boot status
- **Hub <-> Local**: WiFi config, device ready state, account linking, shocker commands
- **HubConfig.fbs**: Persisted device configuration (RF, WiFi, OTA, E-Stop, LAN, etc.), also sent inside `HubToLocalMessage.ReadyMessage`

## Schema Conventions

- **Namespaces**: `OpenShock.Serialization.{Types,Common,Configuration,Gateway,Local}`
- **Message pattern**: Each protocol direction has a root table with an `(fs_serializer)` attribute containing a union payload (e.g., `GatewayToHubMessagePayload`)
- **Shared types**: `Common/` for structures used across protocols (e.g., `ShockerCommand`), `Types/` for enums and small structs
- **Sensitive fields**: `api_key`, `auth_token`, `password` — firmware controls exposure via `withSensitiveData` flag during serialization

## FlatBuffers Compatibility Rules

These are critical — the firmware reads old config buffers after OTA updates:

- **Adding fields**: Always append to end of tables. Never reorder.
- **Default values**: FlatBuffers omits fields matching the default from the binary. Changing a default changes how old buffers with absent fields are read. Only change defaults for fields that weren't previously in use.
- **Unions**: Members are identified by index. Never reorder or remove entries — append only.
- **Enums**: Never change existing values. Append new entries.
- **`(deprecated)` fields**: Keep them in place (removing shifts field IDs).
- **Structs**: Cannot be extended (fixed-size). Use tables if the type may grow.
