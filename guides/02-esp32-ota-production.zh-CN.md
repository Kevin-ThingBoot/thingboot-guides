# 生产级 OTA 升级：A/B 分区、自动回滚与安全模式（ESP32）

> ThingBoot 教程第 2 篇。语言：[English](02-esp32-ota-production.md) | [中文](02-esp32-ota-production.zh-CN.md)

给一块板子用 Wi-Fi 升级固件是 Demo；给现场 10000 台设备升级——
一次失败的更新可能把全部设备变砖——这才是工程。

本篇讲清楚让 OTA 在生产环境安全的三个机制：
**A/B 分区**、**自动回滚**、**兜底安全模式**。

## 为什么"朴素 OTA"会毁掉整个设备群

常见教程的做法是：下载固件 → 覆盖正在运行的程序 → 重启。
如果新固件有 Bug——WiFi 密码错误、崩溃循环、分区表不对——
设备就再也连不回来了。你手上有几千块砖，而且没有物理接触的机会。

生产级 OTA 用一个思想解决：**永远不要覆盖你正在运行的固件**。

## 第 1 步：A/B 分区

ESP32 的 Flash 划分成两个应用槽，由 Bootloader 决定启动哪一个：

```
Flash 布局（A/B）：
[ nvs | otadata | app0 (A) | app1 (B) | spiffs ]

- 正在跑 app0 (A)？ → OTA 把新固件写进 app1 (B)
- 正在跑 app1 (B)？ → OTA 把新固件写进 app0 (A)
- 启动标记（otadata）只在新镜像校验通过后才会切换
```

Arduino 框架下这是 ESP32 的默认行为，`Update` 库会自动写入
**非活动**分区：

```cpp
#include <Update.h>

bool applyOTA(WiFiClient& stream, size_t contentLen) {
  if (!Update.begin(contentLen)) return false;      // 写入另一个槽
  if (Update.writeStream(stream) != contentLen) return false;
  if (!Update.end(true)) return false;              // true = 校验镜像
  ESP.restart();                                    // 重启进入新槽
  return true;
}
```

ESP-IDF 下同样的流程由 `esp_ota_ops.h` 的
`esp_ota_begin()` / `esp_ota_write()` / `esp_ota_end()` 完成。

## 第 2 步：自检通过后才把新镜像标记为"有效"

Bootloader 会照常启动新槽——那新固件坏了怎么办？答案是：
新镜像启动时处于**待确认（pending-validation）**状态，
只有固件自己证明"我活着"之后才转正：

```cpp
// 在新固件里，自检通过后调用：
extern "C" void esp_ota_mark_app_valid_cancel_rollback();
// IDF 中：#include <esp_ota_ops.h>

void setup() {
  initWiFi();
  initMQTT();

  if (selfTestOK()) {
    esp_ota_mark_app_valid_cancel_rollback();  // "这个版本没问题"
  } else {
    ESP.restart();  // 触发下面的回滚机制
  }
}
```

如果 Bootloader 配置了 `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE`，
设备在确认前反复重启达到上限时，**会自动回退到上一个槽**。
整个设备群就能从坏版本中幸存。

> 实操规则：出货前先定义好什么叫"健康"。通常是：
> WiFi 连上 + MQTT 连上 + 第一条遥测数据收到 ACK。

## 第 3 步：安全模式——最后的兜底

即使 A/B + 回滚也可能失效（两个槽都损坏、配置被清空）。
生产设备应保留一个最小"安全模式"镜像：只带 WiFi/MQTT 能力，
等待完整重刷。

常见实现：

- 预留第三个分区放救援镜像
- 通过"开机时按住按键"或"连续 3 次启动失败"进入
- 安全模式下暴露一个 `/recover` HTTP 接口接收固件上传

[ThingBoot SDK](https://github.com/ThingBoot/thingboot-device-esp-arduino-sdk)
把这条链路——A/B 写入、健康确认、回滚、崩溃安全模式——
做成了开箱即用，你只需要配置策略，不用自己移植 Bootloader 逻辑。

## 发布前检查清单

向真实设备推送 OTA 之前：

- [ ] 遥测里带上版本号（你必须*看得见*每台设备跑的哪个版本）
- [ ] 灰度发布：先 1 台 → 10 台 → 100 台，再全量
- [ ] 健康检查在 `mark_valid` 之前执行
- [ ] Bootloader 配置已开启回滚
- [ ] 生产环境镜像签名——防止恶意固件注入
- [ ] 准备好逃生通道（安全模式 / USB 救援）

## 接下来去哪

| 目标 | 教程 |
|---|---|
| [ESP32 发出第一条 MQTT 消息](01-esp32-mqtt-platform.zh-CN.md) | 第 1 篇 |
| 让 AI 智能体通过 MCP 控制设备 | 第 3 篇（计划中） |
| 含配网 + 回滚的完整 SDK | [ThingBoot SDKs](https://github.com/ThingBoot) |

---

*ThingBoot 提供免费商用的设备端 SDK 与私有化 IoT 平台。有问题欢迎提
[issue](https://github.com/Kevin-ThingBoot/thingboot-guides/issues)。*
