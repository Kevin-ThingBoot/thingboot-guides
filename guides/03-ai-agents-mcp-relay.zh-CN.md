# MCP × 单片机：让 Claude 或 GPT 安全地控制一个继电器

> ThingBoot 教程第 3 篇。语言：[English](03-ai-agents-mcp-relay.md) | [中文](03-ai-agents-mcp-relay.zh-CN.md)

第 1 篇让 ESP32 发出了遥测数据，第 2 篇让固件升级不再变砖。本篇闭环最后一块：
**让 AI 智能体对物理设备执行真实动作——而且是在受控的前提下。**

## 缺失的一环：AI 与硬件之间的"共同语言"

你让 ChatGPT"打开水泵"，它会开心地写一首关于水泵的诗——它根本没有通道
触达你的设备。这其实是好事：没有约束的裸访问太危险了。

**MCP（Model Context Protocol）** 就是标准桥梁。AI 不直接碰硬件，而是调用
**MCP Server** 暴露的一小组显式工具：

```
[Claude / GPT / 任意智能体]
        │  MCP（工具调用）
        ▼
[MCP Server]  ← 你的规则都在这：权限范围、人工确认、审计
        │  REST / MQTT
        ▼
[你的平台] → [ESP32 + 继电器]
```

AI 可以*发现*有哪些设备、能做什么，然后*请求*执行——但所有规则
（允许什么、谁确认、记什么日志）都强制在你这一侧执行。

## 第 1 步：把一台设备暴露成 MCP 工具

用 [Open AIoT](https://www.open-aiot.com/zh-cn/) 的参考 MCP Server，
一台符合规范的设备会呈现为可调用的工具，概念上长这样：

```
tools:
  - device_list()                      → "relay-01: 开关, 在线"
  - device_get_state(id)               → {"power": "off"}
  - device_set_property(id, property)  → 例如 power=on  ⚠ 写操作
```

自托管跑起一个 Server（一个容器的事）：

```bash
docker run -d -p 8080:8080 openaiot/mcp-server \
  --backend https://你的平台API \
  --token $平台令牌
```

## 第 2 步：接入你的 AI 客户端

以 Claude Desktop 为例，把 Server 加进 MCP 配置：

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

重启 Claude Desktop，MCP 图标下就能看到设备工具列表——
这就是 AI 通过标准协议"发现"了你的硬件。

## 第 3 步：人在环路的规则（本篇最重要的一段）

现在输入："把 relay-01 打开。"

一个配置正确的系统**不会**直接闭合继电器，而是 AI 陈述意图、
MCP 层要求你确认：

```
Claude 请求调用：device_set_property("relay-01", {"power":"on"})
[仅允许一次] [此设备总是允许] [拒绝]
```

为什么这比看上去重要得多：

- 大模型偶尔会误读语境——误读绝不应该启动一台加热器
- 确认动作天然形成**审计链**：谁发起的、调了什么工具、带了什么参数
- 凭证是带权限范围的（scoped），智能体只能碰你明确共享给它的设备

只读操作（查状态、读遥测）可以自动放行；真实物理输出的写操作
必须人工确认——把这条写进标准，而不是事后打补丁。

## 第 4 步：端到端联测

1. `device_list()` → Claude 看到 relay-01
2. "relay-01 现在什么状态？" → 读操作，返回 power: off
3. "把它打开" → 弹确认框 → 允许
4. 听到继电器"咔哒"一声，再查状态 → power: on

四步走完，你就拥有了一个**在你的规则下**操作硬件的 AI。

## 接下来去哪

| 目标 | 教程 |
|---|---|
| [ESP32 发出第一条 MQTT 消息](01-esp32-mqtt-platform.zh-CN.md) | 第 1 篇 |
| [生产级 OTA：A/B 分区与回滚](02-esp32-ota-production.zh-CN.md) | 第 2 篇 |
| Open AIoT 规范与在线演示 | [open-aiot.com](https://www.open-aiot.com/zh-cn/) |
| 含配网 + 回滚的完整 SDK | [ThingBoot SDKs](https://github.com/ThingBoot) |

---

*ThingBoot 提供免费商用的设备端 SDK 与私有化 IoT 平台。有问题欢迎提
[issue](https://github.com/Kevin-ThingBoot/thingboot-guides/issues)。*
