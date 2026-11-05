# ESP32 → 私有化平台：15 分钟发出你的第一条 MQTT 消息

> ThingBoot 教程第 1 篇。语言：[English](01-esp32-mqtt-platform.md) | [中文](01-esp32-mqtt-platform.zh-CN.md)

大多数 ESP32 教程都停在"点灯"。但真实产品需要的是下一步：
让设备接入一个**你自己拥有**的平台——而不是别人的云。

本篇带你从一块裸 ESP32 开始，通过 MQTT 向自建 Broker 上报数据、接收指令。

## 你需要准备

- ESP32 开发板（ESP32 / ESP32-S3 / ESP32-C3 均可）
- USB 线 + Arduino IDE（或 PlatformIO）
- 一台能跑 MQTT Broker 的机器（Docker 即可）

## 架构

```
[ESP32] --WiFi--> [MQTT Broker（自建）] ---> [你的应用 / 大屏 / AI 智能体]
         <-------- (指令下发 / OTA 触发) ------
```

Broker 自建的好处：数据留在自己网络里、鉴权自己控制，
而且后面 AI 智能体（Agent）可以通过同样的 Topic 直接控制设备。

## 第 1 步：跑起一个 Broker

最快的本地方式：

```bash
docker run -d -p 1883:1883 --name mosquitto eclipse-mosquitto
```

再写一个最小配置 `mosquitto.conf` 用于本地测试：

```
listener 1883
allow_anonymous true
```

> 生产环境务必开启认证 + TLS，见后续《OTA 与安全》篇。

## 第 2 步：给 ESP32 烧录

推荐安装 [ThingBoot ESP Arduino SDK](https://github.com/ThingBoot/thingboot-device-esp-arduino-sdk)，
它把 WiFi 配网、MQTT 断线重连、OTA 都封装进了 setup()/loop() 模型。
也可以用原生 Arduino + PubSubClient，流程完全一样：

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

WiFiClient net;
PubSubClient mqtt(net);

void setup() {
  Serial.begin(115200);
  WiFi.begin("你的WIFI名", "你的WIFI密码");
  while (WiFi.status() != WL_CONNECTED) delay(500);

  mqtt.setServer("192.168.1.100", 1883);   // 改成你 Broker 的局域网 IP
  mqtt.setCallback([](char* topic, byte* payload, unsigned int len) {
    Serial.printf("收到指令 %s: %.*s\n", topic, len, (char*)payload);
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

## 第 3 步：验证

在电脑上订阅，看数据是否到达：

```bash
mosquitto_sub -h 192.168.1.100 -t "things/#" -v
```

再反向给设备发一条指令：

```bash
mosquitto_pub -h 192.168.1.100 -t "things/esp32-demo/cmd" -m '{"relay":"on"}'
```

如果你看到数据在流动、设备能响应指令——恭喜，这就是整个 IoT 设备群的
全部地基：**设备 → 平台 → 设备**。

## 接下来去哪

| 目标 | 教程 |
|---|---|
| 生产环境的 OTA 升级 | 第 2 篇（计划中） |
| 让 AI 智能体通过 MCP 控制设备 | 第 3 篇（计划中） |
| 含配网 + 回滚的完整 SDK | [ThingBoot SDKs](https://github.com/ThingBoot) |

---

*ThingBoot 提供免费商用的设备端 SDK 与私有化 IoT 平台。有问题欢迎提
[issue](https://github.com/Kevin-ThingBoot/thingboot-guides/issues)。*
