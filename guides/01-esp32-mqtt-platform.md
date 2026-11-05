# ESP32 → Self-Hosted IoT Platform: Your First MQTT Message in 15 Minutes

> Part 1 of the ThingBoot Guides. Language: [English](01-esp32-mqtt-platform.md) | [中文](01-esp32-mqtt-platform.zh-CN.md)

Most ESP32 tutorials stop at "blink an LED". Real products need the next step:
get your device talking to a platform you **own** — not someone else's cloud.

This guide walks you from a stock ESP32 to publishing telemetry and receiving
commands over MQTT against a self-hosted broker.

## What you need

- An ESP32 dev board (ESP32 / ESP32-S3 / ESP32-C3 all work)
- USB cable + Arduino IDE (or PlatformIO)
- A machine that can run an MQTT broker (Docker is fine)

## Architecture

```
[ESP32] --WiFi--> [MQTT Broker (self-hosted)] ---> [Your apps / dashboards / AI agents]
         <-------- (commands, OTA triggers) ------
```

Self-hosting the broker means: your data stays on your network, you control
auth, and AI agents can talk to devices through the same topics later.

## Step 1 — Run a broker

The fastest way to get a local broker:

```bash
docker run -d -p 1883:1883 --name mosquitto eclipse-mosquitto
```

Then create a minimal `mosquitto.conf` allowing local testing:

```
listener 1883
allow_anonymous true
```

> For production: always enable auth + TLS. See the OTA & security guide (coming).

## Step 2 — Flash the ESP32

Install the [ThingBoot ESP Arduino SDK](https://github.com/ThingBoot/thingboot-device-esp-arduino-sdk),
which wraps WiFi provisioning, MQTT reconnect, and OTA into a setup()/loop()
model. Or use plain Arduino with PubSubClient — the flow is the same:

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

WiFiClient net;
PubSubClient mqtt(net);

void setup() {
  Serial.begin(115200);
  WiFi.begin("YOUR_SSID", "YOUR_PASSWORD");
  while (WiFi.status() != WL_CONNECTED) delay(500);

  mqtt.setServer("192.168.1.100", 1883);   // your broker's LAN IP
  mqtt.setCallback([](char* topic, byte* payload, unsigned int len) {
    Serial.printf("command received on %s: %.*s\n", topic, len, (char*)payload);
  });
}

void loop() {
  if (!mqtt.connected()) mqtt.connect("esp32-demo");
  mqtt.loop();

  static uint32_t last = 0;
  if (millis() - last > 5000) {
    last = millis();
    mqtt.publish("things/esp32-demo/telemetry",
                 "{\"temp\":25.5,\"rssi\":-60}");
  }
}
```

## Step 3 — Verify

Subscribe from your computer and watch data arrive:

```bash
mosquitto_sub -h 192.168.1.100 -t "things/#" -v
```

Then send a command back to the device:

```bash
mosquitto_pub -h 192.168.1.100 -t "things/esp32-demo/cmd" -m '{"relay":"on"}'
```

If you see telemetry flowing and the device reacting to commands — congratulations,
that's the entire foundation of an IoT fleet: **device → platform → device**.

## Where to go next

| Goal | Guide |
|---|---|
| OTA updates for production fleets | Part 2 (planned) |
| Letting an AI agent control devices via MCP | Part 3 (planned) |
| Full SDK with provisioning + rollback | [ThingBoot SDKs](https://github.com/ThingBoot) |

---

*ThingBoot provides free device SDKs and a self-hosted IoT platform.
Questions? Open an [issue](https://github.com/Kevin-ThingBoot/thingboot-guides/issues).*
