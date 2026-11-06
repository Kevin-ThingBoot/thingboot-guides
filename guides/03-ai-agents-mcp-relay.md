# MCP for Microcontrollers: Letting Claude or GPT Safely Control a Relay

> Part 3 of the ThingBoot Guides. Language: [English](03-ai-agents-mcp-relay.md) | [中文](03-ai-agents-mcp-relay.zh-CN.md)

Part 1 got your ESP32 publishing telemetry. Part 2 made firmware updates safe.
This part closes the loop: **letting an AI agent physically act on your devices —
without giving it unchecked power.**

## The missing piece: a vocabulary between AI and hardware

Ask ChatGPT to "turn on the pump" and it will happily write you a poem about
pumps. It has no way to reach your devices — and that's actually a good thing,
because raw access would be dangerous.

**MCP (Model Context Protocol)** is the standard bridge. The AI doesn't talk to
hardware; it talks to an **MCP server** that exposes a small, explicit set of
tools:

```
[Claude / GPT / any agent]
        │  MCP (tool calls)
        ▼
[MCP Server]  ← your rules live here: scopes, confirmation, audit
        │  REST / MQTT
        ▼
[Your platform] → [ESP32 + relay]
```

The AI can *discover* what devices exist and what they can do, then *request*
actions — but every rule (what's allowed, who confirms, what gets logged) is
enforced on your side of the bridge.

## Step 1 — Expose one device as MCP tools

Using the [Open AIoT](https://www.open-aiot.com/en/) reference MCP server,
a compliant device shows up as callable tools. Conceptually:

```
tools:
  - device_list()                      → "relay-01: switch, online"
  - device_get_state(id)               → {"power": "off"}
  - device_set_property(id, property)  → e.g. power=on  ⚠ write op
```

Run the server (self-hosted, one container):

```bash
docker run -d -p 8080:8080 openaiot/mcp-server \
  --backend https://your-platform-api \
  --token $PLATFORM_TOKEN
```

## Step 2 — Connect your AI client

For Claude Desktop, add the server to the MCP config:

```json
{
  "mcpServers": {
    "open-aiot": {
      "command": "npx",
      "args": ["-y", "@openaiot/mcp-client"],
      "env": { "SERVER_URL": "http://localhost:8080" }
    }
  }
}
```

Restart Claude Desktop. You should now see the device tools listed under the
MCP icon — that's the AI discovering your hardware through the standard.

## Step 3 — The human-in-the-loop rule (this is the important part)

Now type:

> "Turn relay-01 on."

A well-configured setup does **not** flip the relay. Instead, the AI states its
intent and the MCP layer asks you to confirm:

```
Claude wants to call: device_set_property("relay-01", {"power":"on"})
[Allow once] [Always for this device] [Deny]
```

Why this matters more than it looks:

- LLMs misread context occasionally — a misread should never energize a heater
- Confirmation creates an **audit trail**: who asked, what was called, with what
  arguments
- Scoped credentials mean the agent can only touch devices you explicitly
  shared with it

Read-only operations (state, telemetry) can be auto-allowed. Write operations
on real physical outputs should always require confirmation — build that into
the standard, don't patch it on afterwards.

## Step 4 — End-to-end test

1. `device_list()` → Claude sees `relay-01`
2. "What's the state of relay-01?" → read op, returns `power: off`
3. "Turn it on" → confirmation prompt → Allow
4. Watch the relay click; query state again → `power: on`

Four steps, and you have an AI that can operate hardware *under your rules*.

## Where to go next

| Goal | Guide |
|---|---|
| [First MQTT message from ESP32](01-esp32-mqtt-platform.md) | Part 1 |
| [Production-safe OTA](02-esp32-ota-production.md) | Part 2 |
| Open AIoT specification & demo | [open-aiot.com](https://www.open-aiot.com/en/) |
| Full SDK with provisioning + rollback | [ThingBoot SDKs](https://github.com/ThingBoot) |

---

*ThingBoot provides free device SDKs and a self-hosted IoT platform.
Questions? Open an [issue](https://github.com/Kevin-ThingBoot/thingboot-guides/issues).*
