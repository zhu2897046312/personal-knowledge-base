---
title: 智能硬件 Tool Calling 改造构想
tags: [idea, iot, esp32, tool-calling]
created: 2026-08-13
updated: 2026-08-17
aliases: [智能风扇改造, 智能空调改造]
summary: 用 ESP32 + 红外/继电器把家里的风扇、空调等家电改造成能被大模型 Tool Calling 控制的智能设备的项目构想
type: learning
---

# 目标

记录一个还停留在构思阶段的 DIY 项目想法：把家里的哑巴家电（风扇、空调）改造成支持自然语言控制的智能设备，验证"CLI + 本地控制器 + 远程 Go/Eino 云端"这套 Tool Calling 架构在真实硬件上是否可玩。技术细节见 [[LLM Tool Calling 落地：CLI 与配置化 Agent]]，无唤醒词常驻交互见 [[本地端侧意图过滤：无唤醒词常驻 AI 网关]]，本笔记只记录针对具体设备的改造思路、取舍和第一步实操计划。

# 知识点

## 风扇改造（拆机方案）

- 主控用 ESP32（原生支持 Wi-Fi/蓝牙/BLE Mesh，10~15 元）
- 多路继电器模块并联在风扇原机械开关触点上，ESP32 GPIO 驱动继电器模拟"开机/1档/2档/3档/摇头"按键
- ⚠️ 风扇内部是 220V 强电，改线务必断电操作，继电器要带光耦隔离
- 更简单的无损替代方案：
  - **红外遥控改造**（如果风扇带遥控）：ESP32 录制学习原装遥控信号，零安全风险
  - **舵机外挂**：ESP32 控制微型舵机粘在按键上方物理按压，简单粗暴

## 空调改造（红外方案，优先做这个）

比风扇更简单更安全——99% 空调带红外遥控，完全不用拆机、不接触强电。

- 硬件：ESP32 + 红外发射模块即可，成本 10~15 元
- 关键技术细节：空调红外发的是**全量状态包**（开关+模式+温度+风速+扫风一次性打包），不是增量脉冲，所以 CLI 不用记录当前状态，直接把最新全量状态发一次即可
- 开源库 `IRremoteESP8266` 已内置格力/美的/海尔/松下/大金等品牌协议，不需要自己抓包学习编码
- 封装成 `ac-cli control --power=on --mode=cool --temp=26 --fan=low --swing=on` 这样的命令行接口

## 语音识别的分工

- 本地（ESP32）只做离线唤醒词识别（如"小风小风"），几块钱芯片、零延时、不耗流量
- 唤醒后的长语音流通过 WebSocket/HTTP 送云端做 STT（Whisper / 火山引擎等），因为 ESP32 内存跑不了通用 STT 模型

## 第一步实操：先不接大模型，只打通 MQTT → 红外控制空调

优先把"服务端下发 MQTT → 本地控制器解析 → 红外发射控制空调"这条主干通路调通，硬件和协议跑顺后再叠加语音和本地大模型，开发节奏会顺畅很多。

**通信协议**：Topic `home/livingroom/ac/command`，Payload：

```json
{
  "power": "ON",
  "mode": "COOL",
  "temp": 26,
  "fan": "AUTO"
}
```

**本地控制器**（树莓派/香橙派，Python + `paho-mqtt` + Linux `lirc`/`ir-ctl`）：订阅该 Topic，收到消息后解析出 power/temp/mode，调用 `ir-ctl -S gree:POWER_xxx_TEMP_xxx_xxx` 之类的命令发射红外信号。

**Go 服务端下发**（`github.com/eclipse/paho.mqtt.golang`）：构造 `ACCommand{Power, Mode, Temp, Fan}` struct，`json.Marshal` 后 `client.Publish(topic, 0, false, payload)`。

**联调步骤**：本地起一个 Mosquitto/EMQX 容器（`docker run -d --name mosquitto -p 1883:1883 eclipse-mosquitto`）→ 红外发射管接好 GPIO，本地脚本订阅 Topic → 运行 Go 程序触发下发 → 用手机摄像头看红外管是否闪光、空调是否响应 beep 声。通路跑通后，随时可以在这个 Go 服务里接入 Eino / LLM Tool Calling。

## 如果要接入本地大模型做常驻语音过滤：硬件选型要升级

ESP32 单片机算力太弱（RAM 只有几百 KB），跑不动本地 STT 和端侧小模型；ESPHome 也只能生成"纯配置化的简单硬件固件"，做不了这种带 AI 算力的边缘节点。需要换成香橙派/树莓派 5 这类高算力开发板（2~4GB 内存），配 USB 红外学习/发射器 + USB 麦克风。

学习空调红外协议的两种方式：

- **自动抓包解码（推荐）**：用 Linux 的 `LIRC` 或红外解析库，配置里直接选品牌（如格力/美的），自带完整编码算法，不用对着遥控器学。
- **自定义学习**：杂牌空调没有现成协议时，提供 `start_ir_learn` 接口，拿遥控器对红外接收头按一下，采集 GPIO 上的原始脉冲高低电平时间，存成 `ac_on_26_cool` 之类的记录，后续直接重放。

本地节点技术栈：STT 用 `sherpa-onnx` 或 `whisper.cpp`（占用小、出文字快），端侧意图过滤用 Ollama 跑 `Qwen2.5-0.5B`，红外收发用 Linux GPIO/LIRC 库，打包成 Docker 容器跑在开发板上。详细的"无唤醒词常驻过滤"设计见 [[本地端侧意图过滤：无唤醒词常驻 AI 网关]]。

## 造轮子 vs 用现成的：先想清楚目标

- **只想要能用的语音控制环境**：直接上 Home Assistant（Docker 部署）+ ESPHome 刷好 ESP32，HA 后台填 DeepSeek API Key 开启对话代理，一个周末就能全屋落地，完全不需要自己写 Go 服务。
- **想练手、打磨 Go 工程能力，或作为毕业设计/开源项目积累**：Home Assistant 太重且是 Python 写的，值得沿用本笔记这套 Go + Eino + MQTT 的轻量路线，硬件端可以先借力 ESPHome/Tasmota 降低下位机开发成本。

# 示例

体验对比（传统智能家居 vs 接入大模型）：

| 用户说的话 | 生成的 CLI 调用 |
| --- | --- |
| "进屋，好热啊，吹爆！" | `ac-cli control --power=on --mode=cool --temp=18 --fan=high` |
| "有点冷，风直吹得我头疼" | `ac-cli control --power=on --mode=cool --temp=26 --fan=quiet --swing=false` |
| "睡觉了，风别吹太猛，别一直对着我吹" | `fan_set_speed(level=1)` + `fan_set_oscillation(true)` + `fan_set_timer(60)` |

# 总结

> 先从空调红外方案切入（零风险、开源库全帮忙解码），跑通"语音 → 云端 Go/Eino → Tool Calling → ac-cli → 红外发射"全链路后，再考虑风扇的继电器拆机方案。整体架构参考方案 A（本地轻量控制器 + 远程云端大脑），比全本地部署性价比高得多。

# 相关链接

- [[LLM Tool Calling 落地：CLI 与配置化 Agent]]
- [[本地端侧意图过滤：无唤醒词常驻 AI 网关]]
- [[纯软件 AI 项目候选]]
