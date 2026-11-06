# thingboot-guides

Hands-on guides for building real IoT products on ESP32 — the parts vendor
documentation usually skips: provisioning, MQTT that reconnects, OTA that
doesn't brick a fleet, and letting an AI agent control a relay without giving
it unchecked power.

Every guide ships in English and 中文, and every step has been run on real
hardware. The code is standard ESP-IDF / Arduino / MicroPython: it works with
any MQTT broker, and the guides point to the [ThingBoot SDKs](https://github.com/ThingBoot)
where a full provisioning + rollback stack helps.

## Guides

| # | Guide | EN | 中文 |
|---|-------|----|------|
| 1 | Self-hosted ESP32 platform: provisioning, telemetry and remote control | [Read](guides/01-esp32-mqtt-platform.md) | [阅读](guides/01-esp32-mqtt-platform.zh-CN.md) |
| 2 | OTA for production fleets: A/B partitions, rollback, safe mode | [Read](guides/02-esp32-ota-production.md) | [阅读](guides/02-esp32-ota-production.zh-CN.md) |
| 3 | MCP for microcontrollers: letting Claude or GPT control a relay safely | [Read](guides/03-ai-agents-mcp-relay.md) | [阅读](guides/03-ai-agents-mcp-relay.zh-CN.md) |

## Articles

Longer pieces on the engineering and economics behind these guides.

| Article | |
|---------|---|
| [Stop Rebuilding the Same Firmware: Turning One-Off IoT Projects Into Reusable Assets](articles/one-off-iot-projects-to-reusable-assets.md) | Why one-off delivery work never compounds, and what actually makes a codebase reusable across clients |

## Also published on

- DEV Community — https://dev.to/kevin-thingboot
- Medium — https://medium.com/@kevin-thingboot

New guides go up every week or two. Something wrong or missing? Open an
[issue](https://github.com/Kevin-ThingBoot/thingboot-guides/issues).

---

*Maintained by [Kevin](https://github.com/Kevin-ThingBoot) · ThingBoot · Xinbu (Chengdu) Technology Co., Ltd.*
