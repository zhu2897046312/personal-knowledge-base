---
title: 智能硬件 Tool Calling 改造构想
tags: [idea, iot, esp32, tool-calling]
created: 2026-08-13
updated: 2026-08-13
aliases: [智能风扇改造, 智能空调改造]
summary: 用 ESP32 + 红外/继电器把家里的风扇、空调等家电改造成能被大模型 Tool Calling 控制的智能设备的项目构想
type: learning
---

# 目标

记录一个还停留在构思阶段的 DIY 项目想法：把家里的哑巴家电（风扇、空调）改造成支持自然语言控制的智能设备，验证"CLI + 本地控制器 + 远程 Go/Eino 云端"这套 Tool Calling 架构在真实硬件上是否可玩。技术细节见 [[LLM Tool Calling 落地：CLI 与配置化 Agent]]，本笔记只记录针对具体设备的改造思路和取舍。

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
