---
title: LLM Tool Calling 落地：CLI 与配置化 Agent
tags: [tool-calling, agent, go, eino, iot]
created: 2026-08-13
updated: 2026-08-13
aliases: [Tool Calling, 配置驱动 Agent, Dynamic Tool Registry]
summary: 把自己写的 CLI 工具注册成大模型的 Tool，由后端拦截 Tool Call 请求并执行命令；进一步把 Tool 定义做成数据库配置，实现新设备/新工具零代码接入
type: learning
---

# 目标

记录如何让大模型通过 Tool Calling 去执行自己写的 CLI 工具（而不仅仅是回答文本），以及如何把这套机制升级为"配置驱动"的动态 Agent 架构，做到加一个新工具/新硬件不用重新打包部署。

# 知识点

## 1. Tool Calling 调用自定义 CLI 的整体架构

大模型本身不能直接执行命令，只能"决定调用什么命令、用什么参数"，由后端服务作为中介执行：

```text
[ 用户输入 ]
     │
     ▼
1. 传入工具声明（CLI 名称/参数的 JSON Schema 定义）
     │
     ▼
2. LLM 返回结构化 JSON（如 { "tool": "my_cli", "args": [...] }）
     │
     ▼
3. 后端捕获 JSON，用 exec.Command 执行对应 CLI
     │
     ▼
4. 执行结果（stdout/stderr）回传给 LLM 作为上下文
     │
     ▼
5. LLM 总结并回答用户
```

Go 端执行逻辑要点：用 `os/exec` 起子进程，`exec.CommandContext` + `context.WithTimeout` 防止卡死，命令报错时把 stderr 内容回传给大模型让它知道错在哪。

## 2. 生产落地的安全注意事项

1. **白名单校验，禁止拼接任意命令**：绝不能让大模型直接返回一整段 shell 字符串去 `sh -c` 执行（RCE 风险），必须严格限制允许调用的 CLI 白名单，参数走强类型解析。
2. **沙盒运行**：涉及文件修改、编译、系统运维的 CLI 应限制在 Docker 容器或受限权限的用户组内运行。
3. **超时与资源限制**：用 `exec.CommandContext` 配合超时 Context，防止命令卡死拖垮系统。

## 3. 扩展到智能硬件 / 具身智能

把硬件动作（前进、后退、开关、调档）封装成 CLI 或 RPC/HTTP 接口，再让大模型通过 Tool Calling 动态调用，就能把普通硬件变成能听懂自然语言的智能设备——这正是当前智能硬件、具身智能（Embodied AI）、AI 玩具的核心实现范式。

```text
[ 语音识别(STT) ] ──► [ Go/Eino 后端 ] ──Tool Call──► [ 大模型 ]
                            │                              │
                      解析 Tool JSON                  返回调用决策 (如 move_forward)
                            ▼
[ 硬件执行层 (CLI / GPIO / 串口/蓝牙) ]
```

### 家电改造两种思路

- **拆机加装继电器/单片机**（如风扇）：ESP32 + 继电器模块模拟按键，成本低但涉及 220V 强电，需注意光耦隔离安全。
- **红外遥控学习**（如空调，更推荐，零风险）：ESP32 + 红外发射模块，用开源库（`IRremoteESP8266` 等，已内置格力/美的/海尔/大金等品牌协议）直接调用 `setPower/setMode/setTemp/setFan` 生成红外信号，无需拆机、不接触强电。
- **关键细节**：空调红外每次都发送**全量状态包**（开关/模式/温度/风速/扫风一次性打包），不是像风扇/电视那样的单脉冲增量指令，所以 CLI 只需要把大模型生成的最新全量状态一次性发射即可，不用记录当前状态。

### 硬件安全夹具示例

```go
func MoveForward(speed int) error {
    // 强制限制速度范围，防止硬件损坏
    if speed > 100 { speed = 100 }
    if speed < 0 { speed = 0 }
    return exec.Command("./robot-cli", "move", fmt.Sprintf("--speed=%d", speed)).Run()
}
```

需要额外处理**异步中断**：用户执行中途喊"停"，后端要能 Kill 掉正在跑的子进程。

## 4. 架构选型：本地控制器 + 远程云端（推荐）vs 全套本地部署

| 维度 | 方案 A：本地端 + 远程云服务（推荐） | 方案 B：全套本地局域网部署 |
| --- | --- | --- |
| 本地硬件成本 | 十几元（ESP32 单片机） | 几百到几千元（带 NPU/GPU 主板） |
| 断网可用性 | 依赖互联网 | 完全离线可用 |
| 大模型智商 | 极高（云端大模型） | 一般（端侧 1.5B/7B 小模型） |
| 运维难度 | 极低（改云端代码即可全家设备更新） | 较高 |

方案 A 数据流：本地负责唤醒词 + STT 转文字 + 收发 CLI 指令；云端 Go/Eino 服务负责组装 Prompt、调大模型、解析 Tool Call、把指令下发回本地。语音识别建议**本地只做离线唤醒词**（几块钱芯片、零延时），**长语音转文字送云端**（ESP32 内存跑不了通用 STT 模型）。除非对隐私/离线有硬要求，否则不推荐方案 B。

## 5. 配置驱动 Agent：Admin 后台动态注册 Tool

目标：加一个新硬件/新 Tool 时，不用重新写 Go 代码、不用重新打 Docker 镜像，只在 Admin 后台配置即可生效。

核心是组件解耦与动态加载：

```text
Admin 后台新增 Tool 配置（名称/参数 Schema/MQTT 下发主题）
        │ 保存并发布
        ▼
Redis / MySQL（配置中心）
        │ 实时同步 / PubSub
        ▼
远程 Eino Go Server：动态 Tool 注册池 + 通用消息分发器
        │ 下发极简 JSON 指令
        ▼
本地 ESP32 控制器（通用固件，永远不用重刷，只做消息转发）
```

Tool 配置存成 JSON Schema（无需为每个 Tool 写 Go struct）：

```json
{
  "tool_name": "control_heater",
  "description": "控制取暖器开关和温度",
  "parameters_schema": {
    "type": "object",
    "properties": {
      "power": { "type": "string", "enum": ["on", "off"] },
      "temp": { "type": "integer", "minimum": 16, "maximum": 30 }
    },
    "required": ["power"]
  },
  "action_config": {
    "target_device_id": "esp32_living_room",
    "mqtt_topic": "home/living_room/heater/command",
    "payload_template": "{\"action\":\"set\", \"p\": \"{{.power}}\", \"t\": {{.temp}}}"
  }
}
```

Go 服务启动时从数据库读取配置，动态组装 Eino Tool 数组（回调函数逻辑通用，不写 if-else 分支）：

```go
toolsConfig := adminRepo.GetAllEnabledTools()
var einoTools []eino.Tool

for _, cfg := range toolsConfig {
    dynamicTool := eino.NewTool(eino.ToolInfo{
        Name:         cfg.ToolName,
        Description:  cfg.Description,
        ParamsSchema: cfg.ParametersSchema,
    }, func(ctx context.Context, inputJson string) (string, error) {
        payload := renderPayload(cfg.ActionConfig.PayloadTemplate, inputJson)
        mqttClient.Publish(cfg.ActionConfig.MqttTopic, payload)
        return "指令已成功下发给硬件设备", nil
    })
    einoTools = append(einoTools, dynamicTool)
}
```

带来的优势：ESP32 固件彻底"傻瓜化"（只订阅自己的 MQTT Topic 转发 payload，不需要知道自己控制的是什么设备）；新设备接入秒级上架（Admin 勾选保存，Go 服务监听 Pub/Sub 自动热更新 Tool 池，甚至不用重启）；Prompt 与 RAG 参数（Top-K、相似度阈值）也可以放进数据库做热更新。

# 总结

> 从"API 调用师"升级为平台架构师的关键，是把特定业务逻辑抽象成"通用运行引擎 + 动态配置"：用 Go + Eino 做底座引擎，JSON Schema 做动态接口映射，MQTT 传递标准指令，就能搭出一套扩展性无上限的个人智能家居 AI 中枢。

# 相关链接

- [[Eino 与 LangGraph：Chain vs Graph 选型]]
- [[智能硬件 Tool Calling 改造构想]]
