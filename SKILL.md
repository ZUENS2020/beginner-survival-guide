---
name: beginner-survival-guide
description: ZUENS2020's practical coding survival guide — industrial-minimalist approach to problem-solving, with real gotchas from embedded, AI tooling, fuzzing, media, and full-stack projects
metadata:
  tags: beginner, gotchas, embedded, ai-tooling, fuzzing, media, fullstack, troubleshooting, lessons-learned, aesthetic, industrial-minimalism, terminal-first
---

# Beginner Survival Guide

> 真实项目里踩过的坑，提炼成新手避坑指南。
> 不是教科书，是实战笔记。带着我的审美和思维习惯。

---

## 关于作者

这个 Skill 从 ZUENS2020 的公开仓库中提炼而来。作者是一个**在结构化系统中追求精确温度的人**——工业极简主义、终端优先、功能胜过装饰。

**审美底色：** 一个人在庞大、高度结构化的技术系统中，带着精确、克制的温度独自前行。这不是冷峻的虚无，而是那种在高速公路上以稳定速度运转的引擎——不是废墟中摇曳的蜡烛。

**所以这个指南的写法也是：** 不说废话，不装饰，问题 → 解法 → 教训，干净利落。

---

## 我的思维习惯（嵌入在每一条教训里）

读这些教训的时候，你其实也在学我的思考方式：

1. **显式优于隐式** — 永远显式指定，不要依赖默认值。默认值是给 demo 用的，不是给生产用的。
2. **先搞清楚"不能做什么"** — 限制比能力更重要。IMU 测不了平移、BLE 探测不了系统——读文档的 Limitations 章节比读 Features 更有用。
3. **中间件是 bug 高发区** — 任何"翻译层"都是信息丢失的风险点。翻译必须完整，不能只翻译自己理解的部分。
4. **精准比全面重要** — 少而准 > 多而乱。上下文窗口是稀缺资源，不要浪费。
5. **自愈设计** — 永远假设进程会意外终止，设计恢复路径。代理被强杀要能自愈，Agent 记忆要持久化。
6. **不要想当然** — 花 5 分钟读文档，省 5 小时调试。想当然是最贵的 bug。
7. **减法比加法安全** — 拿不准的时候，删掉它。少一个功能比多一个 bug 好。
8. **结构即美感** — 代码的结构、项目的结构、笔记的结构——先把结构搞对，细节自然跟上来。
9. **终端是接口** — CLI 优先于 GUI。数据不需要视觉中介就已经很美了。
10. **精确的温度** — 可以帮你，但不会哄你。尊重你的智商，而不是你的情绪。

---

## 🔌 嵌入式开发（ESP32 / M5Stack Cardputer）

这条线体现的是：**把工业功能主义做到极致**——小小的单功能设备，暴露的 GPIO 引脚就是美，不需要任何装饰。

### ⚠️ 烧录后屏幕没反应？别慌

ESP32-S3 用原生 USB-Serial/JTAG，`pio upload` 结束后的自动复位（RTS pin）在某些情况下——**尤其经 USB Hub 连接时**——会让芯片落进 ROM 下载模式（`boot:0x3 DOWNLOAD`）而不进 app。

**解法：** 按一下机身复位键 / 拔插一次 USB（冷启动）。直连电脑 USB 口通常不会有这个问题。

**教训：** USB Hub 不可靠。嵌入式开发时，直连永远是首选。

### ⚠️ PlatformIO 编译永远带 `-e`

```bash
pio run -e cardputer-adv -t upload   # ⚠️ 永远带 -e
```

不带 `-e` 会编译错误的环境或默认环境，浪费时间排查。

**教训：** 多环境项目里，永远显式指定目标环境，不要依赖默认值。

### ⚠️ IMU 测不了平移位置

加速度二次积分会快速漂移，IMU 只能可靠测**朝向/转角**。

**解法：** 如果要测距离，用 ToF 传感器（VL53L0X），IMU 只负责角度。保持设备大致水平，A→B 在十几秒内完成（无磁力计，yaw 会缓慢漂移）。

**教训：** 传感器选型时先搞清楚"能测什么"和"不能测什么"，不要想当然。

### ⚠️ 表笔别碰 5V / 电池

用 GPIO 做通断测试时，只测断电裸路。碰 5V 或电池会烧 GPIO。

**教训：** 硬件调试前先确认电压范围。3.3V 的 GPIO 承受不了 5V。这是"先搞清楚不能做什么"的经典案例。

### ⚠️ BLE HID 无法探测主机系统

BLE HID 协议没有可靠的方式知道对面是 Mac 还是 Windows。

**解法：** 用 NVS 存一个平台开关，手动切换一次就记住。

**教训：** 协议的局限性要用"用户配置一次"来弥补，不要试图自动检测不可靠的东西。

### ⚠️ pyserial 打开端口前必须 `dtr=True; rts=True`

否则会把 ESP32-S3 推进下载模式。

**教训：** 串口通信的 DTR/RTS 信号线不只是"握手"，它们直接影响芯片启动模式。读文档。

---

## 🤖 AI 工具链（Claude Code / LiteLLM / MCP）

这条线体现的是：**结构即美感**——把 AI 工具链理清楚，中间件就是结构中的关键节点，每个节点都要精确运转。

### ⚠️ Claude Code + LiteLLM 多轮对话会断

用推理模型（DeepSeek-R1 等）通过 LiteLLM 调用时，第二轮开始报错：

```
The `reasoning_content` in the thinking mode must be passed back to the API.
```

**原因：** LiteLLM 的 `/v1/messages` 适配器解析了 Anthropic `thinking` 块，但在序列化出站 OpenAI 请求时丢弃了它们。推理模型拒绝缺少先前推理的多轮历史。

**解法：** 用 [cc-thinkfix](https://github.com/ZUENS2020/cc-thinkfix) 做中间代理，正确翻译 `thinking` ↔ `reasoning_content`。

**教训：** 中间件适配层是 bug 高发区。当两个 API 格式不同时，"翻译"必须完整，不能只翻译自己理解的部分。

### ⚠️ `ANTHROPIC_BASE_URL` 卡在 localhost

如果 `cc-thinkfix` 被 `kill -9` 强杀，`settings.json` 里的 URL 不会自动恢复。

**解法：** 下次启动会通过 sidecar 文件自愈。如果 sidecar 也没了，手动编辑 `settings.json` 恢复真实上游 URL。

**教训：** 代理工具必须能自愈。设计时考虑"如果进程被强杀，用户怎么恢复"。

### ⚠️ AI 生成 OOC（崩人设）——关键词 Lorebook 的问题

传统 Lorebook 靠关键词全量注入上下文，导致 token 浪费且"串词"——AI 写着写着就把人物写歪了。

**解法：** 用图距离驱动的精准上下文注入——只把与当前出场角色 N-hop 内相关的设定喂给 AI。见 [WorldBuilder](https://github.com/ZUENS2020/worldbuilder)。

**教训：** "全量塞给 AI"不是好策略。**精准比全面重要**——这是少而准 > 多而乱的典型案例。上下文窗口是稀缺资源。

### ⚠️ AI Agent 的记忆管理

Agent 跨会话丢失上下文是常态。用 Git 做版本化后端可以提供持久、结构化、可审计的历史。见 [gcc-mem-system](https://github.com/ZUENS2020/gcc-mem-system)。

**教训：** 不要假设 AI 会"记住"任何东西。把重要状态写进持久存储，用 Git 管理版本。**自愈设计**——永远假设进程会意外终止。

---

## 🔒 安全与 Fuzzing

这条线体现的是：**自愈设计和结构即美感**——fuzzing 不是一次性动作，是闭环系统，每个环节都要精确。

### ⚠️ Fuzz 不是"跑一次就完"

单次生成 harness → 跑 fuzzer → 看结果，这只是最浅的一层。真正的 fuzz 工作是闭环：

1. 选目标 → 产脚手架 → 构建（含 coverage 插桩）→ 运行
2. 覆盖率分析 → 改进 harness → 重新构建
3. 崩溃分诊 → 区分 harness bug vs 真实漏洞 → 复现

**教训：** Fuzzing 是迭代过程，不是一次性动作。覆盖率不涨就要改策略，不是加时间。

### ⚠️ 崩溃不等于漏洞

fuzzer 触发的 crash 可能是：
- **harness 本身的 bug**（空指针、越界访问在 harness 代码里）
- **上游库的真实漏洞**
- **不确定**（需要进一步分析）

**解法：** 先分诊（triage），再决定是否上报。不要看到 crash 就兴奋。自己的 bug 先修好，再考虑是不是别人的问题。

**教训：** 分诊是 fuzzing 流程中最容易被跳过、也最不应该被跳过的步骤。**减法比加法安全**——先过滤掉自己的 bug。

### ⚠️ C/C++ 单头库的隐藏陷阱

像 `json.h`、`utf8.h`、`tinyobjloader` 这类单头库看起来简单，但：
- 解析器可能有边界检查缺失（`json_parse_object` 的 OOB）
- "单头"不意味着"安全"，只是"方便集成"
- fuzz 它们时，harness 要覆盖所有解析路径，不只是 happy path

**教训：** 简洁 ≠ 安全。越是看起来简单的代码，越要仔细测试边界条件。**不要想当然。**

---

## 🎬 媒体处理（ffmpeg / 视频导出）

这条线体现的是：**终端是接口**——CLI 优先，ffmpeg 就是完美的终端工具。

### ⚠️ 字幕：软字幕 vs 烧录字幕

- **软字幕（内嵌）**：字幕是独立轨道，可开关，按容器自动适配编码（mp4→mov_text, webm→webvtt）
- **烧录字幕**：字幕直接画在画面上，不可关闭，不可替换

新手经常搞混。如果你要字幕可切换，用软字幕。

**教训：** 先搞清楚你要哪种，再选 ffmpeg 参数。`-c:s copy` 是软字幕，`-vf subtitles=` 是烧录。**先搞清楚不能做什么**——烧录了就回不去了。

### ⚠️ H.264 CRF 值：越小越清晰，但也越大

- High = CRF 16（清晰/大文件）
- Medium = CRF 20（平衡）
- Low = CRF 26（体积优先/画质损失）

**教训：** CRF 不是质量等级，是压缩参数。方向是反的——数字越小质量越高。**不要想当然。** 读文档。

### ⚠️ 动画导出：`requestAnimationFrame` 不可靠

浏览器动画用 `requestAnimationFrame` 驱动，帧间隔不固定。要逐帧精确截图，必须注入虚拟时钟替换 `requestAnimationFrame` / `performance.now` / `Date.now`。

**教训：** 依赖真实时钟的动画无法逐帧控制。要精确导出，必须接管时间。**结构即美感**——要精确控制，先重构时间结构。

---

## 🌐 全栈开发

### ⚠️ Overleaf 文件同步是单向的

AI 修改的文件会推到 Overleaf，但 Overleaf 编辑器里的修改**不会**同步回来。重启会话会从 Overleaf 重新拉取。

**教训：** 双向同步看起来简单，实际上冲突解决极其复杂。先做单向，够用就行。**减法比加法安全。**

### ⚠️ Docker Compose 项目名很重要

```bash
docker compose -p overleaf-claude up -d
```

不带 `-p` 会用目录名，可能导致容器名冲突。

**教训：** **显式优于隐式。** 又是这个教训，但它是通用的。

### ⚠️ frp 隧道：没有域名就只能用 TCP/UDP

HTTP/HTTPS 隧道需要泛域名解析。没有域名就老实用 TCP 隧道。

**教训：** 先确认基础设施条件，再选方案。**先搞清楚不能做什么。**

### ⚠️ 国际化（i18n）不是"翻译一下 UI"

真正的 i18n 涉及三层：
- **Layer A**：文档双语
- **Layer B**：前端 UI 文案（react-i18next，425 个键对齐）
- **Layer C**：后端叙事语言（AI 生成的文本也要跟着语言走）

**教训：** i18n 是全栈问题，不是前端问题。漏掉任何一层都会出现"UI 是英文但 AI 回复是中文"的割裂感。**结构即美感**——三层结构对齐好了，结果自然对。

---

## 🧠 我的思维习惯（总结版）

| 习惯 | 一句话 | 经典案例 |
|------|--------|----------|
| **显式优于隐式** | 别依赖默认值 | Docker Compose 带 `-p`，PlatformIO 带 `-e` |
| **先搞清楚不能做什么** | 读 Limitations 章节 | IMU 测不了平移，BLE 探测不了系统 |
| **中间件是 bug 高发区** | 翻译必须完整 | LiteLLM 丢 thinking 块 |
| **精准比全面重要** | 少而准 > 多而乱 | 图距离注入 vs 关键词 Lorebook |
| **自愈设计** | 假设进程会死 | cc-thinkfix sidecar 恢复 |
| **不要想当然** | 5 分钟省 5 小时 | CRF 方向，传感器能力 |
| **减法比加法安全** | 拿不准就删 | 单向同步先于双向 |
| **结构即美感** | 结构对了细节自然会好 | i18n 三层对齐，fuzzing 闭环 |
| **终端是接口** | CLI 优先于 GUI | ffmpeg 就是最好的媒体工具 |
| **精确的温度** | 尊重智商，不哄情绪 | 这个指南的写法 |

---

## 审美底色（来自 precise-warmth）

这个指南不只是技术教训。它也体现了一种审美态度：

- **工业极简主义** — 每条教训都砍到只剩骨头，不装饰
- **几何秩序** — 结构清晰，问题 → 解法 → 教训，三段式不跑偏
- **克制的调色板** — 只用 ⚠️ 和 **加粗** 做强调，没有花哨的格式
- **终端是界面** — 代码块和命令行是核心表达方式
- **精确的温度** — 可以帮你，但不会哄你。尊重你的智商，而不是你的情绪
- **结构中的能动性** — 新手不是被动的学习者，你是主动在系统中找到方向的人

> 一个人在庞大、高度结构化的技术系统中，带着精确、克制的温度独自前行。
> 这个指南就是你的引擎——稳定、精确、向前。

---

_This skill is auto-maintained. Content distilled from public project experience._
_Last refresh: 2026-07-02_