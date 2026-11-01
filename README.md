# VoxPilot — 智能语音领航员

> 一款面向 AI PC 的桌面环境副驾（Ambient Copilot）。始于语音，不止于语音。

VoxPilot 是一个能听、能看、能动手、也能记住的桌面 Agent。它围绕「手、脑、眼、耳、魂」五个维度构建能力闭环，通过前后端分离的微服务架构，在本地完成低延迟音频处理与推理，同时保留云端能力扩展性。

---

## 核心能力

| 维度           | 能力      | 说明                                                       |
| ------------ | ------- | -------------------------------------------------------- |
| **手 · HAND** | 工具系统    | 云端 Function Calling 执行系统操作；高危工具异步确认，慢任务进程隔离，双向白名单防止幻觉误操作 |
| **脑 · MIND** | 记忆与 RAG | 短期保留最近 N 轮对话；长期用 ChromaDB 存储，后台异步提取实体与向量化                |
| **眼 · EYE**  | 屏幕感知    | 优先 `uiautomation` 提取 UI 文本树，异常降级 `mss` 截图；上传云端需显式确认      |
| **耳 · EAR**  | 系统内录    | WASAPI Loopback 抓取扬声器输出，能量过滤后 `faster-whisper` 转写，触发会议纪要 |
| **魂 · SOUL** | 情绪与主动   | 情感 TTS 动态调节语速音调；EventScheduler 按抑制规则主动播报                 |

---

## 物理架构

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│  VoxPilot-Client            │        │  VoxPilot-Engine            │
│  PyQt6 + C++ / pybind11     │◄──────►│  FastAPI + Python           │
│                             │  WS/gRPC│                             │
│  · UI 渲染与驾驶舱           │        │  · 模型推理 (ASR/LLM/TTS)    │
│  · 音频采集与播放            │        │  · 工具执行与确认            │
│  · FSM 状态机                │        │  · RAG 与长期记忆            │
│  · AudioFocus 音频焦点       │        │  · 多线程下载器              │
└─────────────────────────────┘        └─────────────────────────────┘
```

**架构红线**：

- 🔴 **AEC、VAD、唤醒词必须留在 Client 本地 C++ 层，绝不跨进程**
- 🔒 Engine 默认本地部署（`127.0.0.1`）；远程模式下截图/录音强制弹窗确认，UI 常驻角标，日志脱敏

---

## 落地路线图

| 里程碑    | 目标       | 关键产出                                          |
| ------ | -------- | --------------------------------------------- |
| **M1** | 极简终端闭环   | 录音 5s → faster-whisper → 云端 API → edge-tts 播放 |
| **M2** | UI 与异步调度 | PyQt6 界面，QThread + 信号槽，界面不卡顿                  |
| **M3** | 底层重构与打断  | C++ SPSC 无锁环形缓冲，webrtcvad 断句，Barge-in         |
| **M4** | 手和脑      | Function Calling + 异步确认，Alist 直链下载            |
| **M5** | 微服务化与多模态 | ASR/LLM/TTS 独立 FastAPI 服务，接入视觉                |

> ⚠️ **M3 动手前必须先补并发 pytest**，用测试锁定 SPSC 与确认机制的语义。

---

## 状态机概览

**8 + 1 态**：`IDLE` · `LISTENING` · `ASR` · `LLM` · `TTS` · `PLAYING` · `AMBIENT_MODE` · `PROACTIVE_SPEAKING` · `ERROR`

**核心规则**：

- 任何异常必须归入 `ERROR` 态，防止死循环
- `prev_state_stack` 深度上限 2，栈满拒绝新的主动播报
- 唤醒词是**唯一对话入口**：AMBIENT 态喊唤醒词进 LISTENING，退出内录走快捷键或语音「退出会议模式」
- 音频三分法：麦克风采集永不暂停；ASR 上传在 TTS 时暂停；Barge-in 补传最近 300ms 并恢复上传

---

## 延迟 SLA

| 指标          | 目标          |
| ----------- | ----------- |
| VAD 断句      | < 300ms     |
| ASR 首字      | < 500ms     |
| LLM 首 Token | < 800ms     |
| TTS 首包      | < 300ms     |
| **打断响应**    | **< 150ms** |

> 时钟基准：统一使用 `time.monotonic_ns()`，协议携带 `t_start_ns` 字段保证跨进程可比。

---

## 工程宪法（摘选）

任何代码评审都必须以下列原则为准绳：

1. **GIL 精准释放** — `py::gil_scoped_release` 必须精确包裹数据拷贝循环
2. **契约明确** — `push` 返回 `< len(data)` 表示丢弃需告警；`pop` 返回 `< size` 表示缓冲不足需零填充
3. **TTS 单消费者** — 严禁多线程并发读 stdout；LLM 累积到句号才推入有界队列
4. **工具确认零竞态** — `resolve()` 必须返回是否生效的 `bool`；`confirm_ui` 异常时必须清理 `cid`
5. **慢任务绝对隔离** — 慢工具必须是模块顶层函数，写入 `SLOW_TOOL_REGISTRY`；**严禁把 `self._tools` 传进子进程**
6. **进程启动约束** — `ToolRegistry()` 不得在模块顶层实例化，入口必须有 `if __name__ == "__main__":` 保护

完整 12 条见 `docs/constitution.md`。

---

## 📁 目录结构（规划）

```
VoxPilot/
├── core/                  # 核心逻辑（FSM、工具管理、安全）
│   ├── tools_manager.py   # ToolRegistry + PendingConfirmation
│   └── security.py        # 密钥管理
├── modules/               # 能力模块（网络、视觉、记忆等）
│   └── network/
│       └── alist_client.py
├── cpp_src/               # C++ 底层性能模块
│   └── audio_ringbuf/     # SPSC 无锁环形缓冲
├── ui/                    # PyQt6 界面
├── engine/                # FastAPI 推理服务
├── tests/                 # 并发测试、集成测试
└── main.py                # 入口（须含 __main__ 保护）
```

---

## 📦 环境要求

- Python **3.11+**
- 建议使用 `venv` 隔离环境
- Windows / macOS / Linux（WASAPI 内录仅 Windows）

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

---

## 🔐 隐私与安全

- 默认**本地部署**，模型推理不出本机
- 密钥通过 `keyring` 存储于系统凭据管理器，不落盘、不进代码
- 远程 Engine 模式下，截图与录音上传**强制二次确认**
- 工具系统**双向白名单**：`get_schemas(mode)` 过滤 + `execute` 二次校验 `risk_level`

---

> *“VoxPilot 始于语音，不止于语音。它是一个懂你环境、能动手执行、有记忆的赛博副驾。”*


