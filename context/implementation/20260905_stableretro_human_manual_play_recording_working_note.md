# Working note：手动游玩 interface 与逐帧 trajectory recording

日期：2026-09-05。源码基线：tag `v1.0.1`，commit `ec7a62718a1f99f34bf5e5d5c57255c9a53df507`。

状态：需求、安装说明、代码实现计划和验证计划已整理；本文中的新增 recording 模块、CLI 参数与测试均为待实现项。本地仓库已按要求切到 `v1.0.1`（detached HEAD）；本次交付 working note 和构建并发配置，尚未创建 Conda 环境或安装/实现 recording 功能。

实现决策（2026-09-05 补充）：Stable Retro 保持原始 `v1.0.1` package，采集程序放在包外的 `tools/manual_recording/`，通过现有 API、环境 wrapper 和外部 interface 子类完成需求。此前将 recorder 加入 `stable_retro/`、调整包内 interactive 和增加 package extra 的方案，已统一替换为下文的包外实现计划。Stable Retro 首次安装可能需要编译；之后修改采集程序无需重新编译 emulator。

## 1. 目标与交付范围

提供一个可通过键盘手动玩的窗口，并记录每一次实际 `env.step()` 对应的 image frame、action index、semantic 动作、reward 和 episode 边界。记录可以直接用于 imitation learning、轨迹分析和后续回放校验。

首版以单玩家 `Airstriker-Genesis-v0` 为完整验收对象，自带 ROM，无需额外下载。interface 复用现有 Pyglet 窗口；推荐新建 `stable-retro-recording` 专用环境，并保留 `game-verl` 和 `game-verl-opensrc` 的兼容性验证。其他已有 ROM 的游戏可以复用记录格式，游戏特有动作语义需要另行配置、验证。

结合此前 custom state 需求，首版支持默认 state、integration 内的命名 state，以及外部 gzip `.state` 文件，并保存初始化来源以供复现。多人、浏览器远程控制、音频/视频导出、动作片段合并、随机 state 采样和游玩中热键存档放到后续扩展。

## 2. 当前源码中已确认的行为

| 位置 | 已有能力 / 对计划的影响 |
| --- | --- |
| [interactive.py](../../stable_retro/examples/interactive.py) | `Interactive` 负责事件循环和窗口，`RetroInteractive` 将键盘映射为按钮向量。一次窗口更新可能执行多个 step，因此必须在 step 边界记录，不能按窗口 draw 频率抓图。 |
| [retro_interactive.py](../../stable_retro/examples/retro_interactive.py) | 存在另一份相似入口；包外 interface 复用 `interactive.py` 的 `Interactive` 类，并集中维护自己的按键/语义映射，两份原入口均保持不变。 |
| [retro_env.py](../../stable_retro/retro_env.py) | 默认 `Actions.FILTERED`，action space 是 `MultiBinary`。`action_to_array()` 可能过滤输入；`step()` 返回动作后的 RGB observation、reward、terminated、truncated、info。 |
| [cores/genesis.json](../../cores/genesis.json) | 定义 Genesis 按钮顺序和键盘映射；action index 必须绑定这份按钮顺序。 |
| [playback_movie.py](../../stable_retro/scripts/playback_movie.py) | `.bk2` 回放可以重建动作和画面；现有 CSV 主要记录 episode return，没有实现所需逐步 transition schema。 |
| [determinism.py](../../stable_retro/examples/determinism.py) | 源码明确讨论 core 的确定性限制，以及 Lua scenario、wrapper state 的恢复问题，不能对所有游戏保证无条件逐帧一致。 |
| [Airstriker scenario](../../stable_retro/data/stable/Airstriker-Genesis-v0/scenario.json) | 根据 score 计算 reward；gameover 和 lives 决定终止。日志保存实际返回值，不根据 HUD 或画面推算 reward。 |

包外 interface / wrapper 需要适配的现有细节（无需修改包源码）：

- 当前 `--game` 默认值还没有 `-v0`；新入口显式使用有效 ID。
- `record=None` 会走 `RetroEnv` 的 `record is not False` 分支，从而启用默认目录录制。新入口不录 BK2 时必须传 `record=False`，可选录制时传明确目录。
- 当前 interactive 在 done 后调用 reset，但没有立即更新显示缓存；外部 wrapper 缓存每次 reset/step 的最新 observation，interface 子类在绘制前刷新缓存，让窗口展示新 episode 的初始 observation。
- 当前 `close()` 没有显式调用 `stop_record()`；外部 wrapper 在退出和 episode 切换时明确关闭可选 BK2，再调用原始 env 的清理接口。
- `reset()` 会载入 `initial_state`、清空按键、推进一个 no-op frame，再重置 reward tracking。这个 reset 内部帧不算用户 action step。

环境检查（2026-09-05，仅检查导入和包版本）：两个已有环境均为 Python 3.12、Pillow 12.0.0；`game-verl` 的 Pyglet 为 1.5.0，`game-verl-opensrc` 为 1.5.21，均低于项目要求的 `>=1.5.27,<2`。检查时 checkout 没有编译好的 `_retro` 和 core binaries，两个环境从该目录导入均报 `No module named 'stable_retro._retro'`。切到 tag 后已复核相关 env/interface/recording 源码与此前检查相同；tag 的 `pyproject.toml` 没有 `[dev]` 或 `[recording]` extra。运行验证需要先完成安装；下面没有将任何游戏测试标为已通过。

## 3. interface 行为约定

新入口拟为从仓库根目录执行 `python -m tools.manual_recording.manual_play`；该目录是本地应用代码，不加入 Stable Retro 的 package 清单：

| 参数 | 约定 |
| --- | --- |
| `--game` | 默认 `Airstriker-Genesis-v0`。 |
| `--state NAME` / `--state-file PATH` | 互斥；均不指定时使用 `State.DEFAULT`。外部文件显式 gzip 解压并设置 `initial_state`。 |
| `--scenario` | 默认 integration 的 `scenario`；允许名字或 JSON 路径。 |
| `--record-dir PATH` | 指定时输出 trajectory；省略时只玩游戏。输出目录必须不存在，以独占创建避免覆盖已有记录。 |
| `--record-bk2` | 可选附加 BK2；要求提供 `--record-dir`，每个 episode 单独保存。 |
| `--action-mode all\|filtered` | 新手动入口默认 `all`，保留 Start 等手柄按钮；`filtered` 用于复现现有受限动作行为，必须记录过滤前后差异。 |
| `--semantic-map PATH` | 可选游戏语义 JSON；默认 Airstriker 映射内置，其他游戏退回准确的按钮名称。 |
| `--max-steps N` | 自动化 GUI smoke test 或短录制使用；达到上限只结束采集，不伪造环境返回的 truncated。 |

首版固定 `players=1` 和 `frame_skip=1`，游戏显示采用 `render_mode="rgb_array"`，由现有窗口渲染。运行节奏默认取 core 的 `get_screen_rate()`，同时记录 target rate 与实际 step rate，不能假定所有系统都是精确 60 Hz。

保留现有方向键、Z/X/C、A/S/D、Enter、Tab 等映射；窗口显示当前 game/state、episode/step、action index、semantic action、即时 reward、episode return 和录制状态。`Esc`、关闭窗口和 Ctrl-C 走同一个清理流程。新增 `P` 暂停/恢复、`F9` 手动重新开始 episode、`F1` 显示按键说明；控制热键不进入游戏 action。

失焦时清空按键并暂停，恢复后清空积累的计时误差，避免粘键或补跑大量历史 step。暂停期间没有 action transition；无按键但游戏正常运行时，每帧都记录 NOOP。

## 4. recording 数据协议

### 4.1 时间和图片严格对齐

每一条 transition 的定义固定为：

```text
frame[t] -- effective_action[t] --> frame[t+1], reward[t], terminated[t], truncated[t], info[t]
```

`frame[0]` 来自本 episode 的 `reset()` 返回值；`frame[t+1]` 直接复制本次 `step()` 返回的 observation。保存原始分辨率的 RGB uint8 PNG，不保存窗口缩放结果或 HUD overlay。每个有 N 条 transition 的 episode 必须有 N+1 张 frame，即使两帧像素相同也保留各自编号。

写日志发生在 `step()` 完成之后、下一次 reset 之前。terminal frame 必须保存；下一 episode 独立重新编号。action 是从当前画面到下一画面的输入，不把动作后的图片误当作 decision observation。对照此前只保存 post-action frame 的示例，本 schema 增加了初始帧和 `obs_frame` / `next_obs_frame`，可直接形成训练 tuple。

`step_id` 每 episode 从 0 递增，`global_step` 在 session 内递增，均只统计真正的 `env.step()`。额外保存采样动作时的 `time.monotonic_ns()`，用于分析操作延迟；该字段不能替代 emulator step 编号。

### 4.2 action index 与 semantic 动作

原生 MultiBinary 没有唯一的离散 action ID。首版将日志中的 `action_index` 明确定义为实际执行按钮向量的 bitmask：

```python
action_index = sum(int(bit) << i for i, bit in enumerate(effective_action))
effective_action = [(action_index >> i) & 1 for i in range(num_buttons)]
```

这是本 recording schema 的编码，不是 `retro.Actions.DISCRETE` 的组合索引。需要将 index 还原为按钮向量后再调用默认 MultiBinary 环境；不能直接将该整数传入 `env.step()`。相同 index 的含义依赖 `buttons` 顺序和 mapping version，不保证跨游戏一致。

输入链路为 `keyboard keys → requested_action → action_to_array(requested_action) → effective_action`。环境仍接收 `requested_action` 并按相同模式执行转换；recording 使用转换结果生成 index、按钮名称及语义。在受限模式下保留两份向量，确保日志表达真实送入 emulator 的动作。

当前 Genesis 按钮顺序：

```json
["B", "A", "MODE", "START", "UP", "DOWN", "LEFT", "RIGHT", "C", "Y", "X", "Z"]
```

因此 `NOOP=0`、`B=1`、`LEFT=64`、`B+LEFT=65`。组合动作必须完整保留，不能取最大项或压成单个按钮。

`semantic_action` 分两层保存：

- `pressed_buttons`：按 `buttons` 顺序解码得到的可靠手柄语义，例如 `["B", "LEFT"]`。
- `semantic_action`：基于当前游戏映射的 token 列表，例如 `["fire", "move_left"]`；无人按键为 `["noop"]`。Airstriker 的 `B → fire` 来自现有使用说明，方向键对应移动；其余未验证按钮保留 `button:A` 等形式，不猜测 `jump` / `attack`。JSON 映射复制到 session 中并记录版本/哈希。

语义描述的是发出的操作指令，不表示操作一定成功，例如 `fire` 不保证命中或获得 reward。跨游戏训练如需紧凑统一的 action vocabulary，应在下游转换并保留原始 bitmask。

### 4.3 输出文件与字段

```text
<record-dir>/
  session.json                 # 协议、版本、按钮映射、配置及 session 状态
  assets/                      # 初始化 state、integration JSON、semantic map 等副本
  episodes/
    000000/
      episode.json             # state 来源、return、计数、end_reason、结束状态
      transitions.jsonl        # 一行一个 env.step()
      frames/
        00000000.png           # reset 后的初始 observation
        00000001.png           # step 0 之后的 observation
        ...
      recording.bk2            # 可选
    000001/
      ...
```

下面是 schema 示例，数值仅用于说明，不是实测轨迹。frame 路径相对当前 episode 目录：

```json
{
  "schema_version": 1,
  "episode_id": 0,
  "step_id": 0,
  "global_step": 0,
  "input_time_ns": 123456789000,
  "obs_frame": "frames/00000000.png",
  "next_obs_frame": "frames/00000001.png",
  "keyboard_keys": ["LEFT", "X"],
  "requested_action": [1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0],
  "effective_action": [1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0],
  "action_index": 65,
  "pressed_buttons": ["B", "LEFT"],
  "semantic_action": ["fire", "move_left"],
  "reward": 0.0,
  "episode_return": 0.0,
  "terminated": false,
  "truncated": false,
  "info": {"score": 0, "lives": 3, "gameover": 0}
}
```

`reward` 保存未经 clipping、scaling、帧聚合的实际返回值；`episode_return` 是本 episode 中这些 reward 的累计和。保留零值和负值；将 NumPy scalar/array 转为合法 JSON 类型，禁止悄悄字符串化或丢字段。首版单玩家 reward 为标量。

`session.json` 至少包含 game ID、integration 类型、system/core 名称及二进制哈希、Stable Retro 版本/基线 commit、包外采集工具 commit 或源码哈希及 dirty 状态、Python/NumPy/Gymnasium/Pyglet/Pillow 版本、ROM 实际哈希及算法、scenario/data/metadata 副本与哈希、state 来源、buttons 顺序、keyboard mapping、semantic mapping、action encoding/version、action mode、players、frame shape/dtype、frame_skip、core FPS、启动时间和完成状态。不复制 ROM。工具可以与上游源码位于同一仓库，但工具版本与 Stable Retro 基线要分别记录；未提交的工具文件不能仅用仓库 HEAD 标识。

若 scenario 使用 Lua 脚本，需要将其依赖的 integration 脚本一并保存并记录哈希；无法收集完整依赖时显式标记 replay 所需的缺失资源。episode summary 保存 `num_steps`、`num_frames`、`return`、`end_reason`，并区分正常终止、手动 reset、user_exit、max_steps 和 error。用户提前退出可以完成数据落盘，但该 episode 标记为 interrupted；不要把最后一条 transition 的环境终止标志改成 true。

### 4.4 写入性能、错误与退出

主线程独占 emulator 和 Pyglet，生成 image/action/info 的独立快照；单独 writer 使用有界 FIFO 队列写 PNG 和 JSONL，保持顺序。必须复制 NumPy 像素缓冲和可变 info，避免后续 step 改写已排队数据。队列初始上限建议 120 帧，配置与峰值计入性能报告。

先完整写入 PNG 临时文件并 rename，再提交引用它的 JSONL 行；JSONL 定期 flush，episode 结束必定 flush。队列满则阻塞生产并显示写入压力，不能静默丢帧；慢磁盘下允许实际游戏速度降低。writer 异常需传回主线程，停止后续 step，确保等待中的生产者和 shutdown 不会死锁。

正常退出时停止输入/step、结束 episode、排空队列、关闭 BK2 和文件、更新 summary，最后释放 env/window。磁盘满等异常将 session 标为 error，尽力保存可恢复的已提交前缀，不标记 success。强制 kill 可能留下末尾不完整行或孤立 PNG，校验工具应明确报告；不承诺操作系统崩溃后的完整持久性。

推荐录到节点本地 `/tmp` 或任务 scratch，再打包拷回 GPFS：约 60 FPS 下每分钟约 3600 张 PNG。需记录输出位置并由用户在节点/作业释放前搬走数据。

## 5. custom state 和 replay 约定

命名 state 使用 `make(state=...)` / `load_state()`；外部 state 使用 `gzip.open(..., "rb")` 加载到 `env.initial_state`，并在第一次 reset 前设置。新入口不提供无 snapshot 的开机模式；如果默认 state 缺失，应提示提供命名或外部 state，避免 `State.NONE` 后续 reset 无法恢复起点的问题。

session 保存用于 reset 的原始 `initial_state` 解压字节的哈希及 gzip 副本，记录它是 reset 的输入，随后会推进一个 no-op frame。不要将 reset 后 `em.get_state()` 的 snapshot 当作相同输入，否则回放再 reset 会额外前进一帧。外部 state 的 `statename` 需设置为稳定非空标签，以兼容 BK2 文件命名。

同 ROM/core/state/config 下 reset 的画面和 RAM 应在目标游戏上验证一致。`reset(seed=...)` 不会自动选择不同 state；不同 core、版本或外部 wrapper 的 RNG/计时器不在 emulator snapshot 的普遍保证范围内。

主要记录在手动游玩时直接生成：RGB、实际 action、reward、done、info 是当时运行的观测事实。BK2 可选，用于压缩动作存档和回放交叉检查。离线校验重建相同初始化和 scenario，然后逐步比较这些记录；出现差异时报告首个 mismatch，不覆盖原始采集 reward。BK2 不能替代所需的 scenario/版本信息，也不能证明所有游戏的 replay 一定完全确定。

## 6. 代码实现计划

### 6.1 已有 API 与包外职责

现有 `v1.0.1` 已提供模拟游戏及采集所需的底层数据，但没有一个现成 CLI 可以输出本文全部 PNG/JSONL 字段。新增 Python 应用负责组织这些调用和落盘。

| 需求 | 直接调用的已有接口 | 包外程序需要补充的逻辑 |
| --- | --- | --- |
| 创建游戏 / 命名 state | `retro.make(state=...)`、`env.load_state()`、`env.reset()` | CLI 参数校验，选择起点并保存初始化来源。 |
| 图片 / reward / done / info | `env.reset()`、`env.step(action)` 的返回值 | recorder wrapper 保存初始帧和每个 transition，维持严格对齐及 episode 边界。 |
| 实际执行按钮 | `env.action_to_array(action)` | 保存 requested/effective 两份向量，按第 4.2 节编码 action index。 |
| 按钮语义 | `env.get_action_meaning(action)`、`env.buttons` | 得到 LEFT/B 等按钮名，再使用游戏映射生成 move_left/fire；index 和游戏语义不由包自动生成。 |
| custom state | `env.em.get_state()`、`env.initial_state`、`env.statename` | gzip 存取、命名、哈希、reset 前加载；state 格式和 core 兼容性校验。 |
| BK2 | `env.auto_record(path)`、`env.stop_record()`、`retro.Movie` | 每 episode 的文件管理、显式结束录制、回放核验；保持原始 reset/step 的 Movie 写入顺序。 |
| 手动窗口 | `stable_retro.examples.interactive.Interactive(env=...)` | 外部子类接收 wrapper，提供按键映射、HUD、暂停/失焦和退出管理。 |

原生 env 在完成 state/scenario 配置后包装为 `RecordingEnv(gymnasium.Wrapper)`，再传给外部 `ManualPlayInterface(Interactive)`。wrapper 覆盖 `reset()` / `step()` / `close()`，调用底层接口并记录，不改返回的 Gymnasium tuple。访问 Stable Retro 特有接口时显式用 `wrapped_env.unwrapped`，避免依赖 Gymnasium 对未知属性的隐式转发。键盘键名和采样时间通过 wrapper 的 input context setter 在 step 前传入，headless 测试可以直接提供相同上下文。

必须在 `Interactive.__init__()` 第一次调用 reset **之前**完成 recorder/wrapper 初始化。这里复用的是已经接受 `env` 的基类 `Interactive`，不调用会自行创建裸 env 的 `RetroInteractive.__init__()`。HUD、暂停和生命周期行为在外部子类覆盖必要方法；不 monkey-patch `retro.make` 或 package 方法。基类来自 examples，其窗口缓存等内部属性依赖固定的 `v1.0.1`，升级包时需要重新验证。

### 6.2 拟新增目录

```text
tools/
  __init__.py
  manual_recording/
    __init__.py
    manual_play.py             # CLI 和 ManualPlayInterface
    recorder.py                # RecordingEnv、TrajectoryRecorder、writer
    action_mapping.py          # 键盘映射、index codec、游戏语义
    validate_recording.py      # schema 校验和 replay
    requirements.txt           # 应用自身的依赖；Stable Retro 固定使用本地 v1.0.1
    README.md                  # 应用启动及日志说明
    tests/
      test_recording_actions.py
      test_recording.py
      test_manual_play_integration.py
      test_manual_play_ui.py
```

这些文件位于 Stable Retro 的 Python package 之外。`stable_retro/`、`retro/`、`src/`、`cores/`、上游 `setup.py` / `pyproject.toml` 和原有 examples/tests 都不需要修改。构建并发仍通过已有外部 `build_recording.cfg` 传入。Pillow 等额外依赖写入工具自己的 requirements，不为上游 package 增加 `[recording]` extra。

### 6.3 分阶段落地

| 阶段 | 拟新增位置（相对 `tools/manual_recording/`） | 工作与完成标准 |
| --- | --- | --- |
| P1：数据协议和动作 | `action_mapping.py` | 在工具内集中定义兼容原入口的键盘映射、纯函数 bitmask codec、effective action 解码、Airstriker 语义及自定义映射校验；未配置语义有明确 fallback。 |
| P2：采集和写入 | `recorder.py` | 实现 `RecordingEnv`、`TrajectoryRecorder` 与 writer；拦截 reset/step/close，实现 begin/end episode、初始帧和逐步落盘、summary/config/state、错误传播及幂等关闭。该模块不导入 Pyglet。 |
| P3：手动入口 | `manual_play.py` | 创建裸 env、设置 state/scenario，再包装并注入现有 `Interactive`；外部子类补充 HUD、暂停、失焦、F9 和统一退出，在绘制前同步 wrapper 的最新图像；每次 reset/step 只调用一次。 |
| P4：附加 BK2 | `recorder.py` / `manual_play.py` | 默认裸 env 使用 `record=False`；需要 BK2 时调用现有 `auto_record()`，episode 切换前调用 `stop_record()`，配置新目录后再 reset。校验首尾帧和动作偏移，退出显式关闭 Movie。 |
| P5：校验和 replay | `validate_recording.py` | 默认只检查 schema、引用、计数、编码和 return；`--replay` 逐 episode 用保存的 state/config 与有效动作验证 RGB/reward/done/info。支持 `--check-bk2`，失败返回非零并报告 episode/step/字段。 |
| P6：依赖、文档和测试 | `requirements.txt`、`README.md`、`tests/` | 描述专用环境和工具运行方式；单独声明 Pillow 等应用依赖；完成下述验收并记录工具版本和上游基线。修改这些 Python 文件无需重新安装或编译 Stable Retro。 |

UI 负责将按键转换为 requested action 并调用 wrapper；wrapper 的 `step()` 内部记录顺序如下（伪代码，`base_env` 是原始 RetroEnv，`recorder` 是写入组件）：

```python
effective = base_env.action_to_array(requested)[0].copy()  # 首版单玩家
next_obs, reward, terminated, truncated, info = base_env.step(requested)
recorder.record_step(
    requested_action=requested,
    effective_action=effective,
    next_obs=next_obs.copy(),
    reward=reward,
    terminated=terminated,
    truncated=truncated,
    info=info,
)  # recorder 内部保存 keyboard/time 上下文和前一 frame 引用，并复制可变字段
if terminated or truncated:
    recorder.end_episode(reason="environment_done")
    # 先结束当前 episode/BK2，再 reset 并保存新 episode 的 frame[0]
```

UI 的初始化、自动 reset 和 F9 手动 reset 必须统一走 wrapper 的生命周期接口；首次 reset 就要经过 recorder，不能先丢掉初始帧再开始记录。wrapper 的 step 将原始 tuple 返回 UI，由 UI 在 done 后调用 wrapped env 的 reset；wrapper 不在 step 内自动 reset，避免重复执行或覆盖 terminal frame。手动 reset 先标记 end_reason，然后 reset 创建新 episode。recording 的处理耗时不调用额外 emulator step。

## 7. 验证计划和验收标准

### 7.1 环境准备

推荐使用 `stable-retro-recording` 专用环境收集数据，后续训练只读取 PNG/JSONL。该环境不需要 Torch、VERL、CUDA、Stable Baselines3 或 ffmpeg；只有后续另做视频导出才需要 ffmpeg。2026-09-05 已按本节步骤创建环境并完成本地 tag 的安装；实测补充依赖和验收结果见下文及 7.5。

**创建环境和准备 Python 依赖：**

```bash
conda create -n stable-retro-recording python=3.12 pip -y
conda activate stable-retro-recording
cd /scratch/gpfs/CHIJ/xinran/projects/game-envalgm/Stable-Retro

git describe --tags --exact-match HEAD
# 预期 v1.0.1；在后续实现有提交后另记录其 commit

python -m pip install "setuptools==79.0.1" wheel build "cmake>=3.10,<4"
python -m pip install "numpy>=1.26,<2" "gymnasium>=1.0.0" \
    "pyglet>=1.5.27,<2" pillow pytest
```

这里选择 Python 3.12、NumPy 1.x 作为首个验证组合；固定 setuptools 79.0.1 是为了使构建配置解析与已检查的版本一致。CMake 采用 3.x 建立首个源码构建基线。其余依赖在安装验证后导出精确版本，不能将上述范围依赖当作 lockfile。

登录节点已有系统 GCC/G++、Make、pkg-config、zlib/libzip headers；计算节点的开发依赖不一定相同，需在实际编译节点检查：

```bash
command -v gcc g++ make cmake pkg-config
pkg-config --modversion zlib libzip
```

如果迁移到缺少这些工具/headers 的计算节点，可加载集群提供的开发工具 module；没有合适 module 时，在专用环境中补充 Conda 工具链：

```bash
# 仅在系统开发依赖不足时执行
conda install -n stable-retro-recording -c conda-forge \
    c-compiler cxx-compiler make pkg-config zlib libzip
conda deactivate
conda activate stable-retro-recording
export CMAKE_PREFIX_PATH="$CONDA_PREFIX${CMAKE_PREFIX_PATH:+:$CMAKE_PREFIX_PATH}"
export PKG_CONFIG_PATH="$CONDA_PREFIX/lib/pkgconfig${PKG_CONFIG_PATH:+:$PKG_CONFIG_PATH}"
```

2026-09-05 在 Della CPU 节点实测还需要补充以下依赖。即使设置 `BUILD_N64=OFF` 和 `ENABLE_HW_RENDER=OFF`，此 tag 仍默认构建需要 OpenGL headers 的 DS/melonDS core；系统 GCC 11.5 也缺少项目静态链接所需的 `libstdc++.a`。无需修改上游源码，在新环境补齐即可：

```bash
conda install -n stable-retro-recording -c conda-forge \
    libgl-devel libglu "libstdcxx-devel_linux-64=11.4.0" -y
conda activate stable-retro-recording

# 仅给当前构建 shell 提供依赖搜索路径，不修改其他 Conda 环境。
export CPATH="$CONDA_PREFIX/include${CPATH:+:$CPATH}"
export LIBRARY_PATH="$CONDA_PREFIX/lib/gcc/x86_64-conda-linux-gnu/11.4.0:$CONDA_PREFIX/lib${LIBRARY_PATH:+:$LIBRARY_PATH}"
```

此处 GCC 11 的静态开发库路径对应本次 Della 系统编译器；换编译器时需重新检查，不能机械沿用。GLU 用于 Pyglet 窗口运行。构建提示缺少 CapnProto 时禁用的是 RAM search 结果的保存/加载，不是 emulator save state；本次不为该非录制功能补装 CapnProto。

**从本地 tag 安装，限制编译并发：**

随 note 提供 [build_recording.cfg](build_recording.cfg)，内容为 `[build_ext] parallel = 4`。通过 setuptools 的 `DIST_EXTRA_CONFIG` 加载后，项目 `setup.py` 的 `self.parallel` 将驱动 `make -j4`。已在现有 setuptools 79.0.1 中验证初次及重新初始化 build_ext 都解析出 4；这项配置检查没有执行编译。将 4 调整为不超过实际分配的 CPU 数，并在计算节点执行；当前源码的默认值是整台机器的 `multiprocessing.cpu_count()`，仅设置 `CMAKE_BUILD_PARALLEL_LEVEL` 并不能覆盖它。

```bash
DIST_EXTRA_CONFIG="$PWD/context/implementation/build_recording.cfg" \
STABLE_RETRO_CMAKE_ARGS="-DBUILD_TESTS=OFF -DBUILD_UI=OFF -DBUILD_N64=OFF -DENABLE_HW_RENDER=OFF" \
    python -m pip install --no-build-isolation -e .

python -m pip check
python -c "import stable_retro as r; print(r.__version__); print(r.__file__)"
```

`--no-build-isolation` 使用上面已经准备的构建依赖。`BUILD_UI=OFF` 只禁用 Qt integration editor，Pyglet 手动玩窗口仍可用；Airstriker 不需要 N64 或 Dreamcast 硬件渲染。当前 tag 没有 `recording` extra，本方案也不添加它；Pillow 独立安装，P6 将采集工具依赖整理进 `tools/manual_recording/requirements.txt`。该文件实现后使用 `python -m pip install -r tools/manual_recording/requirements.txt` 安装工具依赖，不重装上游 package。

如果必须同时在多个环境中编译 editable 安装，应使用各自的源码 checkout，避免共享 in-place CMake cache 和二进制被交叉覆盖。首版首先完成专用环境安装和验收，再顺序检查两个已有环境，记录各自实际加载的包路径；新环境成功不代表旧环境自动兼容。

**无窗口 smoke test（当前 tag 已支持这些 API）：**

```bash
python - <<'PY'
import numpy as np
import stable_retro as retro

env = retro.make(
    game="Airstriker-Genesis-v0", state="Level1",
    render_mode="rgb_array", record=False,
)
try:
    initial, _ = env.reset()
    initial = initial.copy()
    print("buttons:", env.buttons)
    print("observation:", initial.shape, initial.dtype)
    action = np.zeros(env.num_buttons, dtype=np.int8)
    total_reward = 0.0
    for i in range(120):
        obs, reward, terminated, truncated, info = env.step(action)
        total_reward += reward
        if terminated or truncated:
            break
    restored, _ = env.reset()
    print("steps:", i + 1, "return:", total_reward, "info:", info)
    print("same reset pixels:", np.array_equal(initial, restored))
    assert np.array_equal(initial, restored)
finally:
    env.close()
PY
```

**窗口试玩和依赖留档：**

```bash
# 需要当前节点上可用的 X11 forwarding 或 VNC；不能仅靠随意设置 DISPLAY
python -m stable_retro.examples.interactive --game Airstriker-Genesis-v0

# 在安装和 smoke test 通过后，保存到新的专属目录
retro_env_export_dir=$(mktemp -d /tmp/stable-retro-recording-env.XXXXXX)
conda env export --no-builds > "$retro_env_export_dir/environment.yml"
python -m pip freeze > "$retro_env_export_dir/requirements-lock.txt"
git rev-parse HEAD > "$retro_env_export_dir/source-commit.txt"
```

当前旧入口的 `record=None` 行为可能在当前目录生成 BK2；第 2 节的适配由包外入口完成，原始示例保持原样。上面的 smoke test 显式关闭录制，新入口按第 3 节的开关约定验收。Pyglet 窗口仍需要系统 X11/OpenGL/GLU runtime，缺失时通过集群 module/管理员提供；Airstriker 的无窗口 smoke 不需要 X Server。

将依赖留档复制进正式 session 的 assets/config 目录；`pip freeze` 中 editable 安装的本地路径不能代替 source commit。完成代码后，先在专用环境运行下述全套验证，再在 `game-verl` 和 `game-verl-opensrc` 运行兼容性检查。GUI smoke 用 Xvfb，人工体验通过 X11/VNC；Xvfb 通过只证明窗口生命周期能运行，不代表真实键盘手感已验收。

### 7.2 自动验证矩阵

| 验证项 | 方法 | 通过标准 |
| --- | --- | --- |
| action index / semantic | `test_recording_actions.py`：遍历 Genesis 12-bit 向量、不同按钮顺序、NOOP、组合键、unknown semantic；另用受限 action fixture | index 与向量双向一致；65 精确对应 B+LEFT；语义从 effective action 生成；过滤前后差异被保留；Start 和未知按钮不静默消失。 |
| transition 对齐 | `test_recording.py`：fake env 返回像素值等于 step 的图像，并复用/改写同一缓冲区；一轮 draw 驱动多个 step | N 行对应 N+1 张正确图；每条连接 t 与 t+1；初始帧、最后一帧、相同像素帧都存在；异步写入不会保存成后续帧。 |
| reward 与 episode 边界 | fake env 包含零/正/负 reward，分别返回 terminated、truncated；触发 F9、max_steps、提前退出 | return 等于逐步 reward 和；terminal 图先于 reset 保存；新 episode 的 ID/return 重置；没有伪造环境 done 或将 reset 算作 action。 |
| 文件生命周期和失败 | 临时目录、重复输出路径、慢 writer、队列满、写入异常、关闭两次 | 正常 close 后所有提交行可读且引用存在；拒绝覆盖；无丢帧/死锁；异常返回非零并标记 error；仅恢复前缀时明确报告未完成。 |
| custom state | `test_manual_play_integration.py`：Airstriker 命名 state、运行后保存的 gzip state、重复 reset、不同 seed | 同 state 的初始 RGB/RAM 一致；custom state 实际被加载；与直接加载加一次 no-op 的参考相符；错误 gzip/缺失文件清晰报错。 |
| 真实 emulator 记录一致性 | Airstriker 从同 state 执行固定 600 步输入序列（NOOP、移动、组合开火）；先 baseline，关闭 env 后再 recorder run | 按对应 step 比较 RGB、reward、terminated/truncated、info；录制不改变结果。若提前 done，固定该边界并独立开启下一 episode。不要同进程同时创建两个 emulator。 |
| 原始日志 replay | 对上一行产物执行 `validate_recording --replay`，包括 custom state session | RGB 按解码后的像素比较；done 和离散 info 精确匹配；reward 使用明确记录的数值容差（例如绝对误差 1e-6）；报告首个差异。 |
| 可选 BK2 | 同 session 开启 BK2，显式关闭后重新加载并逐步比对；覆盖第二个 episode 和 custom state | 文件可读；action 次数、有效向量和首尾 RGB 与直接日志对齐；没有 reset header 偏移；reward 在相同 scenario 下符合容差。 |
| GUI 事件 | `test_manual_play_ui.py`：在 Xvfb 下测试组合键/松键/暂停/恢复/失焦/退出及多步 update | 事件影响下一模拟 step；NOOP 被记录；暂停无 step；无粘键；done 后显示新初始帧；退出完整落盘。 |
| 包外边界 | 安装原始 `v1.0.1` 后，从根目录启动工具；核对上游受保护源码与 tag 的 diff，比较更改工具前后的 core 哈希 | 无需修改/monkey-patch 上游 package，工具修改后直接运行，无新增 core 编译；env 类、action 转换、reward 和 state 恢复仍使用原始实现。 |

上述新增测试均放在 `tools/manual_recording/tests/`。纯 codec / writer 用 fake env 测试以避免 GUI 和商业 ROM 依赖；真实测试仅依赖随仓库提供的 Airstriker。若 Python package 导入仍需要 `_retro`，应先完成构建而不是将 runtime 缺失当成测试成功。特别验证 recorder 在第一次 reset 前安装、每个 wrapper step 只调用一次底层 step，以及需要 core API 的代码显式使用 `unwrapped`。

结构校验还应检查 JSON 类型、路径均落在 episode/session 内、连续 step 编号、有效向量长度、index 解码、semantic map 一致性、PNG 的 shape/dtype、相邻 frame 引用连续，以及 summary 与日志一致。可选保存逐帧原始 RGB SHA-256 加速后续完整性核验；PNG 文件二进制差异不等于像素差异。

### 7.3 人工验收和性能

连续玩至少 5 分钟，覆盖方向移动、单键开火、移动+开火、松开后的 NOOP、暂停/恢复、失焦、F9、环境自然结束和 Esc/关闭窗口。核对 HUD 中 index/semantic/reward 与同一步日志一致，抽查起始/中间/末尾图片和总 return。尚未自然触发 done 时，不能仅靠 F9 宣称真实终止流程已经人工通过。

在本地临时磁盘运行固定序列的录制开/关对照，记录 step 吞吐、p50/p95 step 加入队列耗时、队列峰值、阻塞时间、退出 drain 时间及磁盘占用。功能硬要求是 0 静默丢帧和准确对齐；目标是在选定计算节点上保持 core 的实时速度。若达不到，报告具体速度/瓶颈，优先调整 PNG 压缩和写入方式，不通过减少日志采样掩盖问题。GPFS 性能单独报告。

### 7.4 实现后的预期运行命令

以下命令依赖 P1–P6 完成，目前不是已存在的 CLI。测试先在 `stable-retro-recording` 运行，再在已安装所需依赖的 `game-verl` / `game-verl-opensrc` 顺序复测；新 session 使用新的输出目录。

```bash
conda activate stable-retro-recording
cd /scratch/gpfs/CHIJ/xinran/projects/game-envalgm/Stable-Retro

python -m pytest -q tools/manual_recording/tests/test_recording_actions.py tools/manual_recording/tests/test_recording.py tools/manual_recording/tests/test_manual_play_integration.py
xvfb-run -a python -m pytest -q tools/manual_recording/tests/test_manual_play_ui.py

python -m tools.manual_recording.manual_play \
    --game Airstriker-Genesis-v0 --state Level1 \
    --record-dir /tmp/airstriker-manual-session-001 --record-bk2

python -m tools.manual_recording.validate_recording \
    /tmp/airstriker-manual-session-001 --replay --check-bk2

# custom state 验收：替换为实际已经保存的 gzip state 路径
python -m tools.manual_recording.manual_play \
    --game Airstriker-Genesis-v0 --state-file /path/to/custom.state \
    --record-dir /tmp/airstriker-custom-session-001
```

验收结果须记录日期、commit、环境、执行命令、输出目录、step/frame 数、replay 是否一致及性能数据。P1–P6 录制工具及其完整验证矩阵仍待实现和执行；已完成的环境和原始 emulator 测试见 7.5，不能视为 recorder 验收通过。

### 7.5 2026-09-05 实际安装与基础验收

- 环境：`/home/xl9353/.conda/envs/stable-retro-recording`；Python 3.12.14。
- Stable Retro：1.0.1，commit `ec7a62718a1f99f34bf5e5d5c57255c9a53df507`，editable 指向本仓库。没有修改两个已有 Conda 环境的安装包；共享源码现在包含专用环境构建的 native 二进制，不能据此推断旧环境的 ABI 兼容性。
- Python 依赖：NumPy 1.26.4、Gymnasium 1.3.0、Pyglet 1.5.31、Pillow 12.3.0、CMake 3.31.10、setuptools 79.0.1；完整 Python 依赖版本见 [requirements 快照](20260905_stable_retro_recording_requirements.txt)。Conda 另外提供 OpenGL/GLU 和 GCC 11.4 静态 C++ 开发库。
- 编译：Slurm CPU 作业 `13499208`，4 CPU / 8 GiB，成功完成；此前作业 `13499123` 因缺少开发库报错后已取消。构建日志为 `context/implementation/stable-retro-recording-build.log`。共生成 11 个 core；本次关闭 N64、硬件渲染、Qt UI 和 C++ tests，没有更改上游 package 实现。
- `pip check`、包导入、版本/数据路径检查：通过。
- 7 个游戏各 120 步无窗口抽测：通过。检查实际 ROM hash、RGB/dtype、reward、PNG 编码、默认 state 重复 reset 和运行中保存的 custom state 重复 reset。详细 ROM 覆盖及测试结果见 [ROM 验证记录](20260905_rom_validation.md)。未批量正式导入 ROM，测试使用临时 custom integrations。
- 官方 `RetroInteractive` 在 Xvfb 中创建窗口、模拟和绘图 60 步、关闭：通过。真实 `DISPLAY=localhost:10.0` 无法连接；需恢复有效 SSH X11 forwarding 或在 VNC 中启动。未验收真实键盘手感。
- 本次没有实现逐帧 image/action/reward JSONL recorder，也没有执行 recorder replay、BK2 对齐或性能验收。

在 Della 重现源码构建时，先按 7.1 激活环境并设置 `CPATH` / `LIBRARY_PATH`，再提交小型 CPU 作业：

```bash
srun --partition=cpu --nodes=1 --ntasks=1 --cpus-per-task=4 \
    --mem=8G --time=01:00:00 --job-name=stable-retro-build \
    env DIST_EXTRA_CONFIG="$PWD/context/implementation/build_recording.cfg" \
    STABLE_RETRO_CMAKE_ARGS="-DBUILD_TESTS=OFF -DBUILD_UI=OFF -DBUILD_N64=OFF -DENABLE_HW_RENDER=OFF" \
    python -m pip install --no-build-isolation --no-deps --no-index -e .
```

这里 `--no-deps --no-index` 的前提是 7.1 的 Python 依赖已装好，使计算节点编译阶段无需访问包索引。源安装为 in-place/editable；将来切换源码 tag 或 Python 版本时应重新核验/构建，不能移动或删除此仓库后仍期望该环境独立运行。
