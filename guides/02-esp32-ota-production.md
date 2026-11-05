# OTA for Production Fleets: A/B Partitions, Rollback, and Safe Mode on ESP32

> Part 2 of the ThingBoot Guides. Language: [English](02-esp32-ota-production.md) | [中文](02-esp32-ota-production.zh-CN.md)

Shipping firmware over Wi-Fi to one board is a demo. Shipping to 10,000 boards
in the field — where a bad update can brick them all — is engineering.

This guide covers the three mechanisms that make OTA safe for production:
**A/B partitions**, **automatic rollback**, and a **last-resort safe mode**.

## Why naive OTA kills fleets

A typical "OTA tutorial" does this: download firmware, write over the running
application, reboot. If the new firmware has a bug — bad WiFi credentials,
a crash loop, a wrong partition table — the device never comes back online.
You now have thousands of paperweights and no physical access.

Production-grade OTA solves this with one idea: **never overwrite the firmware
you are running**.

## Step 1 — A/B partitions

ESP32's flash is split into two application slots. The bootloader decides
which one to boot:

```
Flash layout (A/B):
[ nvs | otadata | app0 (A) | app1 (B) | spiffs ]

- Running from app0 (A)?  → OTA writes new firmware to app1 (B)
- Running from app1 (B)?  → OTA writes new firmware to app0 (A)
- Boot flag (otadata) only flips after the new image passes validation
```

In Arduino framework this is the default for ESP32 — the `Update` library
targets the *inactive* partition automatically:

```cpp
#include <Update.h>

bool applyOTA(WiFiClient& stream, size_t contentLen) {
  if (!Update.begin(contentLen)) return false;      // writes to the OTHER slot
  if (Update.writeStream(stream) != contentLen) return false;
  if (!Update.end(true)) return false;              // true = validate image
  ESP.restart();                                    // boots into new slot
  return true;
}
```

In ESP-IDF, the same flow is handled by `esp_ota_begin()` / `esp_ota_write()` /
`esp_ota_end()` from `esp_ota_ops.h`.

## Step 2 — Mark the image valid only after a health check

The bootloader will happily boot the new slot — so how do you roll back if the
new firmware is broken? The image starts in a **pending-validation** state.
Only after your firmware proves itself does it confirm:

```cpp
// Inside the NEW firmware, after self-tests pass:
extern "C" void esp_ota_mark_app_valid_cancel_rollback();
// in IDF: #include <esp_ota_ops.h>

void setup() {
  initWiFi();
  initMQTT();

  if (selfTestOK()) {
    esp_ota_mark_app_valid_cancel_rollback();  // "this version works"
  } else {
    ESP.restart();  // will trigger rollback below
  }
}
```

If the device reboots `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` times without
confirming, **the bootloader automatically falls back to the previous slot**.
Your fleet survives the bad release.

> Practical rule: define what "healthy" means before you ship. Usually:
> WiFi connected + MQTT connected + first telemetry ack received.

## Step 3 — Safe mode: the last resort

Even A/B + rollback can fail (both slots corrupted, config wiped). Production
devices should keep a minimal "safe mode" image: a tiny firmware that only
brings up WiFi/MQTT and waits for a full re-flash.

Common implementations:

- Reserve a third partition with the rescue image
- Boot into it via a button held at power-on, or 3 consecutive failed boots
- In safe mode, expose a `/recover` HTTP endpoint that accepts a firmware upload

The [ThingBoot SDK](https://github.com/ThingBoot/thingboot-device-esp-arduino-sdk)
implements this chain — A/B write, health-check confirmation, rollback, and a
crash safe mode — out of the box, so you configure policy instead of porting
bootloader logic.

## Deployment checklist

Before you push an OTA release to real devices:

- [ ] Version number in telemetry (you must *see* which fleet runs what)
- [ ] Staged rollout: 1 → 10 → 100 devices before fleet-wide
- [ ] Health check runs before `mark_valid`
- [ ] Rollback enabled in bootloader config
- [ ] Signed images (production) — prevents malicious firmware injection
- [ ] A known-good escape hatch (safe mode / USB rescue)

## Where to go next

| Goal | Guide |
|---|---|
| [First MQTT message from ESP32](01-esp32-mqtt-platform.md) | Part 1 |
| Letting an AI agent control devices via MCP | Part 3 (planned) |
| Full SDK with provisioning + rollback | [ThingBoot SDKs](https://github.com/ThingBoot) |

---

*ThingBoot provides free device SDKs and a self-hosted IoT platform.
Questions? Open an [issue](https://github.com/Kevin-ThingBoot/thingboot-guides/issues).*
