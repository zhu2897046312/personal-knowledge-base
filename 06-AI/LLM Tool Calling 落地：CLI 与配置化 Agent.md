---
title: LLM Tool Calling 落地：CLI 与配置化 Agent
tags: [tool-calling, agent, go, eino, iot]
created: 2026-08-13
updated: 2026-08-17
aliases: [Tool Calling, 配置驱动 Agent, Dynamic Tool Registry, DeviceCommand]
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

## 5. 双向数据链路：不只是"下发指令"，还要"状态上报"

完整链路是 `空调 ↔ 本地控制器（ESP32/树莓派） ↔ Go Eino 服务（本地或云端） ↔ 大模型 API`，容易只关注"从上往下发指令"的正向流程，而忽略状态/传感器数据要反向上报：

- **正向（控制指令链）**：用户说话 → Go Eino 服务调大模型 → 大模型返回 Tool Call → 服务通过 WebSocket/MQTT 把 JSON 指令发给本地控制器 → 本地控制器发射红外/蓝牙驱动硬件。
- **反向（状态上报链）**：本地控制器接了温湿度传感器（如 DHT11）等，定时把 `{"current_temp": 30, "humidity": 75}` 上报给 Go Eino 服务更新内存中的环境状态；当温度超过阈值时，服务甚至不用等用户开口，就能让大模型主动建议："检测到室内温度 30℃，是否为你开启空调？"

Go Eino 服务放在本地局域网（树莓派/NAS）还是远程云端各有取舍：云端方案最省钱省心、改 Prompt/Tool 只需重新部署云端包，但本地控制器断网就无法工作；本地局域网部署延迟更低、断网仍可用，但需要一台常年开机的设备，运维成本更高。绝大多数个人 DIY 优先选云端。

## 6. 把大模型输出的 JSON 清洗成 MQTT 需要的 JSON

大模型 Tool Call 输出的是"意图抽取结果"，硬件需要的是特定协议的 Payload，中间是一层数据映射与格式化。三种实现方案：

- **方案一：`text/template` 模板引擎**（推荐，适合 Admin 动态配置）：MQTT Payload 结构写成模板存数据库，Go 服务用标准库渲染，不改代码即可支持新设备。渲染前用 `json.Unmarshal` 解析 LLM 输出为 `map[string]any`，渲染后再 `json.Unmarshal` 校验结果是合法 JSON。
- **方案二：Struct 强类型映射**（适合固定设备、追求极致性能）：定义 `LLMACInput` / `MQTTAcPayload` 两个 struct，写清晰的字段级转换函数（含枚举值归一化，如 `fan_speed: "high"` → `Spd: 3`）。
- **方案三：通用映射字典 + JSON 路径提取**（如 `tidwall/gjson` + `tidwall/sjson`）：适合只需要改字段 key 名的简单场景，用 `map[string]string` 描述"LLM 字段路径 → MQTT 字段路径"的映射规则。

无论用哪种方案，清洗层都必须做 3 个防护：**数值范围裁剪**（大模型误返回超范围值时钳制，如温度>30 强制设为 30）、**默认值兜底**（字段缺失时给安全默认值）、**类型强制转换**（防止大模型把数字返回成字符串）。

## 7. Admin 配置化与强类型绑定：通用 DeviceCommand 模型

"配置驱动"和"强类型安全"并不矛盾，反而应该绑定在一起才是生产级实现。几乎所有家用电器的控制，抽象到底层都只是三类强类型参数的组合：**开关量**（Boolean/Enum，如开关/摇头）、**连续数值量**（Int/Float，如温度/风速/定时）、**模式离散量**（Enum，如制冷/制热/除湿）。

```go
package device

// DeviceCommand 通用硬件控制指令（强类型）
type DeviceCommand struct {
    DeviceID string                 `json:"device_id"`
    Action   string                 `json:"action"`
    Power    *bool                  `json:"power,omitempty"`
    Mode     string                 `json:"mode,omitempty"`
    Value    *float64               `json:"value,omitempty"` // 温度/风速/亮度等连续量
    Extra    map[string]interface{} `json:"extra,omitempty"`
}
```

Admin 后台配置的不再是任意 JSON，而是"标准动作 → 设备底层协议"的映射规则（如 `payload_format: {"pwr": "{{if .Power}}1{{else}}0{{end}}", "temp": "{{.Value}}"}`）。处理链路：大模型输出 JSON → Go 用 `DeviceCommand` 强类型反序列化并校验范围（超范围自动钳制，如 `math.Max(16, math.Min(30, *cmd.Value))`）→ 读取该 DeviceID 对应的 Admin 模板配置 → 模板渲染出最终 MQTT Payload → 下发。

这样带来三个优势：**绝对安全防护**（强类型 + 范围校验杜绝大模型吐出乱码烧坏硬件）、**全家电通用**（空调/风扇/台灯/扫地机共用一套 struct 和模板引擎）、**Admin 界面标准化**（前端甚至能直接根据这个强类型结构体渲染配置表单）。

## 8. 造轮子还是用轮子：开源生态里已有的成熟方案

把系统抽象到这个层级时，其实已经在重新推导目前最成熟的开源智能家居架构：

- **Home Assistant（HA）**：全球最大的开源智能家居平台，底层就是"强类型域（Domain，如 `light`/`climate`）+ 设备强制归一化"，社区的 Extended OpenAI Conversation 等插件做的正是"设备列表动态生成 JSON Schema 注册给大模型，大模型输出后拦截执行标准 Service Call"这一套逻辑。缺点是用 Python 写、体量庞大。
- **ESPHome**：C++ 开源固件生成器，写几十行 YAML 定义引脚功能，编译烧录即得到带 MQTT/OTA 的固件，不用手写单片机代码。但它只能做"纯配置化的简单硬件固件"，跑不了本地 STT/端侧小模型（见 [[本地端侧意图过滤：无唤醒词常驻 AI 网关]]），这种更重的边缘 AI 节点需要香橙派/树莓派 5 这类高算力开发板自己写程序。

选型建议：**只想要能用的智能家电环境** → 直接 Home Assistant + ESPHome 拼装，几天落地；**想练手打造轻量级 Go 引擎、做毕业设计或开源项目积累** → 沿用本笔记的 Go + Eino + MQTT 路线，硬件端可以先借力 ESPHome/Tasmota，云端大脑自己写。

# 总结

> 从"API 调用师"升级为平台架构师的关键，是把特定业务逻辑抽象成"通用运行引擎 + 动态配置"：用 Go + Eino 做底座引擎，JSON Schema 做动态接口映射，MQTT 传递标准指令，就能搭出一套扩展性无上限的个人智能家居 AI 中枢。造轮子前先看一眼 Home Assistant + ESPHome，想清楚是要"能用"还是要"练手"。

# 相关链接

- [[Eino 与 LangGraph：Chain vs Graph 选型]]
- [[智能硬件 Tool Calling 改造构想]]
- [[本地端侧意图过滤：无唤醒词常驻 AI 网关]]
