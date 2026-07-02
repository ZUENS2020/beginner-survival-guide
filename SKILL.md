---
name: beginner-survival-guide
description: Practical lessons and gotchas distilled from real projects — helps beginners avoid common pitfalls in embedded, AI tooling, fuzzing, media processing, and full-stack development
metadata:
  tags: beginner, gotchas, embedded, ai-tooling, fuzzing, media, fullstack, troubleshooting, lessons-learned
---

# Beginner Survival Guide

> 真实项目里踩过的坑，提炼成新手避坑指南。
> 不是教科书，是实战笔记。

这个 Skill 从公开仓库的实际开发经历中提炼而来。每一条都是有人真的踩过、花过时间才搞明白的。目的是让刚学编程的人少走弯路。

---

## 🔌 嵌入式开发（ESP32 / M5Stack Cardputer）

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

**教训：** 硬件调试前先确认电压范围。3.3V 的 GPIO 承受不了 5V。

### ⚠️ BLE HID 无法探测主机系统

BLE HID 协议没有可靠的方式知道对面是 Mac 还是 Windows。

**解法：** 用 NVS 存一个平台开关，手动切换一次就记住。

**教训：** 协议的局限性要用"用户配置一次"来弥补，不要试图自动检测不可靠的东西。

### ⚠️ pyserial 打开端口前必须 `dtr=True; rts=True`

否则会把 ESP32-S3 推进下载模式。

**教训：** 串口通信的 DTR/RTS 信号线不只是"握手"，它们直接影响芯片启动模式。读文档。

---

## 🤖 AI 工具链（Claude Code / LiteLLM / MCP）

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

**教训：** "全量塞给 AI"不是好策略。精准比全面重要。上下文窗口是稀缺资源。

### ⚠️ AI Agent 的记忆管理

Agent 跨会话丢失上下文是常态。用 Git 做版本化后端可以提供持久、结构化、可审计的历史。见 [gcc-mem-system](https://github.com/ZUENS2020/gcc-mem-system)。

**教训：** 不要假设 AI 会"记住"任何东西。把重要状态写进持久存储，用 Git 管理版本。

---

## 🔒 安全与 Fuzzing

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

**解法：** 先分诊（triage），再决定是否上报。不要看到 crash 就兴奋。

**教训：** 分诊是 fuzzing 流程中最容易被跳过、也最不应该被跳过的步骤。

### ⚠️ C/C++ 单头库的隐藏陷阱

像 `json.h`、`utf8.h`、`tinyobjloader` 这类单头库看起来简单，但：
- 解析器可能有边界检查缺失（`json_parse_object` 的 OOB）
- "单头"不意味着"安全"，只是"方便集成"
- fuzz 它们时，harness 要覆盖所有解析路径，不只是 happy path

**教训：** 简洁 ≠ 安全。越是看起来简单的代码，越要仔细测试边界条件。

---

## 🎬 媒体处理（ffmpeg / 视频导出）

### ⚠️ 字幕：软字幕 vs 烧录字幕

- **软字幕（内嵌）**：字幕是独立轨道，可开关，按容器自动适配编码（mp4→mov_text, webm→webvtt）
- **烧录字幕**：字幕直接画在画面上，不可关闭，不可替换

新手经常搞混。如果你要字幕可切换，用软字幕。

**教训：** 先搞清楚你要哪种，再选 ffmpeg 参数。`-c:s copy` 是软字幕，`-vf subtitles=` 是烧录。

### ⚠️ H.264 CRF 值：越小越清晰，但也越大

- High = CRF 16（清晰/大文件）
- Medium = CRF 20（平衡）
- Low = CRF 26（体积优先/画质损失）

**教训：** CRF 不是质量等级，是压缩参数。方向是反的——数字越小质量越高。

### ⚠️ 动画导出：`requestAnimationFrame` 不可靠

浏览器动画用 `requestAnimationFrame` 驱动，帧间隔不固定。要逐帧精确截图，必须注入虚拟时钟替换 `requestAnimationFrame` / `performance.now` / `Date.now`。

**教训：** 依赖真实时钟的动画无法逐帧控制。要精确导出，必须接管时间。

---

## 🌐 全栈开发

### ⚠️ Overleaf 文件同步是单向的

AI 修改的文件会推到 Overleaf，但 Overleaf 编辑器里的修改**不会**同步回来。重启会话会从 Overleaf 重新拉取。

**教训：** 双向同步看起来简单，实际上冲突解决极其复杂。先做单向，够用就行。

### ⚠️ Docker Compose 项目名很重要

```bash
docker compose -p overleaf-claude up -d
```

不带 `-p` 会用目录名，可能导致容器名冲突。

**教训：** 显式指定项目名，不要依赖默认值（又是这个教训）。

### ⚠️ frp 隧道：没有域名就只能用 TCP/UDP

HTTP/HTTPS 隧道需要泛域名解析。没有域名就老实用 TCP 隧道。

**教训：** 先确认基础设施条件，再选方案。不要在"假设有域名"的前提下设计。

### ⚠️ 国际化（i18n）不是"翻译一下 UI"

真正的 i18n 涉及三层：
- **Layer A**：文档双语
- **Layer B**：前端 UI 文案（react-i18next，425 个键对齐）
- **Layer C**：后端叙事语言（AI 生成的文本也要跟着语言走）

**教训：** i18n 是全栈问题，不是前端问题。漏掉任何一层都会出现"UI 是英文但 AI 回复是中文"的割裂感。

---

## 🧠 通用教训

### 1. 显式优于隐式

多环境项目带 `-e`，Docker Compose 带 `-p`，串口带 `dtr=True`——**永远显式指定，不要依赖默认值**。默认值是给 demo 用的，不是给生产用的。

### 2. 先搞清楚"不能做什么"

IMU 测不了平移、BLE 探测不了系统、没有域名不能用 HTTP 隧道——**限制比能力更重要**。先读文档的"Limitations"章节。

### 3. 中间件是 bug 高发区

API 翻译层、协议适配器、代理工具——任何"中间人"都是信息丢失的风险点。**翻译必须完整，不能只翻译自己理解的部分。**

### 4. 精准比全面重要

上下文注入用图距离不用关键词全量、fuzzing 分诊后再上报、i18n 三层对齐——**少而准 > 多而乱**。

### 5. 自愈设计

代理被强杀后要能恢复、Agent 记忆要持久化、Git 做版本化后端——**永远假设进程会意外终止，设计恢复路径。**

### 6. 不要想当然

传感器能测什么、协议能探测什么、字幕是软的还是硬的——**想当然是最贵的 bug**。花 5 分钟读文档，省 5 小时调试。

---

_This skill is auto-maintained. Content distilled from public project experience._
_Last refresh: 2026-07-02_