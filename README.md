
下面给你一套**能直接开干**的“桌面多模态感知智能体助手”技术栈 + 项目框架（含目录结构、模块职责、数据流、依赖建议）。我按“可稳定交付 + 好写报告 + 易扩展”来设计。

---

## 技术栈（推荐组合）

### 1) 语言与运行

* **Python 3.10/3.11**

### 2) 视觉感知（核心）

* **MediaPipe**

  * `FaceMesh`：眨眼/闭眼、打哈欠、头部朝向（粗略）
  * `Hands`：手势（OK/握拳/张开手掌）
  * （可选）`Pose`：活动状态/健身计数
* **OpenCV (cv2)**：摄像头采集、画面叠加、帧处理

### 3) 桌面 UI（强烈推荐）

* **PySide6（Qt for Python）**（比 PyQt5 更推荐，生态/授权更舒服）

  * 主窗口 + 状态面板 + 设置页
  * 托盘（System Tray）+ 通知弹窗

> 如果你想极快做出来：可以先用“OpenCV 窗口 + 键盘控制”做 MVP，但桌面助手更建议 PySide6。

### 4) 语音（可选加分）

* 输入（语音指令）：

  * 离线：**Vosk**
  * 在线：SpeechRecognition + Google（不稳定/需网络）
* 输出（语音提醒）：

  * **pyttsx3**（离线 TTS，最稳）
  * 或 edge-tts（需要网络）

### 5) 数据与日志（加分、也利于报告）

* **SQLite（内置）**：存事件、会话、统计
* 或先用 **CSV/JSONL**（更简单）
* 可视化（加分）：**matplotlib** 生成“今日专注/疲劳曲线”

### 6) 工程化

* **Poetry 或 venv + requirements.txt**
* **loguru**：漂亮的日志
* **pydantic**：定义感知结果/事件结构（让代码更像“系统”）

---

## 项目框架（目录结构 + 每个文件做什么）

建议项目名：`focusmate_desktop_assistant`

```
focusmate_desktop_assistant/
├─ README.md
├─ requirements.txt
├─ main.py
├─ config/
│  ├─ default.yaml
│  └─ user.yaml
├─ assets/
│  ├─ icons/
│  └─ sounds/
├─ app/
│  ├─ __init__.py
│  ├─ bootstrap.py              # 启动装配：加载配置、初始化组件
│  ├─ runtime.py                # 应用主循环协调（UI<->Agent）
│  └─ constants.py
├─ perception/                  # 多模态“感知层”
│  ├─ __init__.py
│  ├─ camera.py                 # OpenCV 摄像头读取 + 帧率控制 + 线程
│  ├─ face.py                   # FaceMesh -> blink/yawn/head_pose
│  ├─ hands.py                  # Hands -> gesture (OK/FIST/PALM/...)
│  ├─ pose.py                   # 可选：Pose -> activity/squat counter
│  ├─ smoothing.py              # 平滑滤波/防抖（EMA/滑窗）
│  └─ schemas.py                # Pydantic: PerceptionState 等结构
├─ agent/                       # “智能体层”：记忆 + 策略 + 状态机
│  ├─ __init__.py
│  ├─ memory.py                 # 事件队列、滑动窗口统计、最近状态
│  ├─ policy.py                 # 规则策略：何时提醒/如何提醒
│  ├─ state_machine.py          # 模式：Study/DND/Meeting/Fitness
│  ├─ agent.py                  # 核心 loop: perceive->update->decide->act
│  └─ schemas.py                # Event/Action/AgentState 数据结构
├─ actions/                     # “执行层”
│  ├─ __init__.py
│  ├─ notifier.py               # 桌面通知/弹窗（Qt）
│  ├─ tts.py                    # 语音播报（pyttsx3）
│  ├─ hotkeys.py                # 可选：触发系统快捷键（音量/媒体键）
│  └─ logger.py                 # 写入 CSV/SQLite
├─ ui/                          # “界面层”
│  ├─ __init__.py
│  ├─ main_window.py            # 主窗口（摄像头预览 + 状态卡片）
│  ├─ widgets.py                # 状态卡、阈值滑条等
│  ├─ tray.py                   # 托盘菜单（启停、模式切换、退出）
│  ├─ settings_dialog.py        # 设置弹窗：阈值、提醒频率
│  └─ resources.qrc             # Qt 资源（可选）
└─ data/
   ├─ focusmate.db              # SQLite（运行后生成）
   └─ exports/                  # 导出的报告/CSV
```

---

## 核心数据流（你照这个写，报告也好写）

**Camera Frame**
→ `perception.face` / `perception.hands` 得到结构化状态 `PerceptionState`
→ `agent.memory` 更新滑动窗口（最近 5 分钟事件、疲劳指数、专注指数）
→ `agent.policy` 输出 `Action`（提醒/延后/切模式/语音）
→ `actions.notifier/tts/logger` 执行
→ `ui.main_window` 实时显示（状态+事件+提示）

---

## 关键“结构化数据”设计（建议你就按这个实现）

### PerceptionState（每帧或每秒更新）

* `presence: bool`（是否检测到脸）
* `blink_rate: float`（眨眼频率/分钟）
* `eye_closed_sec: float`（闭眼持续时间）
* `yawn: bool`
* `head_dir: Literal["center","left","right","down"]`
* `gesture: Literal["none","ok","fist","palm"]`
* `activity: Literal["still","moving"]`（可选）

### Event（写入 memory 与数据库）

* `type`: `"LOOKING_AWAY" | "DROWSY" | "YAWN" | "LEFT_SEAT" | "GESTURE_OK" ...`
* `ts`: 时间戳
* `confidence`: 0-1
* `meta`: 附加信息（持续秒数、方向等）

### Action（policy 输出）

* `type`: `"NOTIFY" | "SPEAK" | "SNOOZE" | "SWITCH_MODE" | "NOOP"`
* `payload`: 文本、持续时间等

---

## 依赖建议（requirements.txt 示例）

（按常见可用版本写，你装的时候如果冲突再微调）

* `mediapipe`
* `opencv-python`
* `numpy`
* `PySide6`
* `pydantic`
* `loguru`
* `pyttsx3`（可选）
* `matplotlib`（可选，做报告图）
* `pyyaml`

---

## MVP 的“最小实现路径”（保证你能跑起来）

1. `ui/main_window.py`：先做 Qt 窗口 + 两个标签（摄像头画面、状态文本）
2. `perception/camera.py`：摄像头线程输出帧
3. `perception/face.py`：FaceMesh 跑通，能输出 `presence` + `eye_closed_sec`（先别做全部）
4. `agent/agent.py`：每秒取一次 PerceptionState，policy 里先写 2 条规则：

   * 闭眼 > 2s → NOTIFY + SPEAK
   * head_dir != center 持续 > 10s → NOTIFY
5. `actions/notifier.py`：先用 UI 弹一个提示条（别急着做系统通知）
6. 再加 `hands.py`：OK 手势触发 `SNOOZE(10min)`

---

## 你可以直接写进报告的“创新点/亮点”

* 多模态融合：Face + Hands（+可选 Pose/Voice）
* 结构化感知 → 事件化记忆（滑动窗口）
* 策略层解耦（Policy 可替换：规则/LLM）
* 桌面端工程化：UI、托盘、配置、日志、数据持久化

---

如果你愿意，我下一条可以直接给你：

* 一份**最小可运行代码骨架**（main.py + PySide6 窗口 + 摄像头线程 + FaceMesh 接入）
* 以及 `PerceptionState / Event / Action` 的 pydantic 定义模板
  你想要用 **PySide6** 还是 **PyQt5**？（我默认 PySide6，最推荐）
