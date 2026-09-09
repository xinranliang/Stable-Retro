# Working note：手动游玩 interface 与逐帧 trajectory recording

日期：2026-09-05。源码基线：tag `v1.0.1`，commit `ec7a62718a1f99f34bf5e5d5c57255c9a53df507`。

最新状态（2026-09-09，America/New_York）：包外采集工具支持 `--append` + `--stop-on-done` 多次启动、自动接续 episode 编号；已按用户选择直接沿用 `--state`，新增 session/run/episode 初始化关卡日志，完整自动化回归 146 项通过。第 8 节为 append 实现，第 9 节保留当时只读探索，第 10 节记录初始化日志实现与验证；不新增 `--world` / `--level`。第 11 节新增多-state 游戏名称资产：68 个游戏的 JSON + 汇总索引，此次非 GUI 回归 149 项通过、13 项 GUI 未运行。后续 Stable Retro recording 的 implementation working note 统一补充在本文件；采集工具代码和使用/schema 文档仍保留在独立工具仓库。以下 2026-09-05/06 内容保留为历史记录。

状态（2026-09-06 更新）：专用环境和包外采集工具已完成，默认在录制时保存逐帧 emulator state；68 项自动化测试通过，长时真实桌面人工验收仍待完成。最新实现、命令、数据字段和验证边界见 [2026-09-06 实现总结](20260906_stableretro_human_recording_implementation_summary.md)。

本文定位：保留 2026-09-05 的设计过程和当时的安装记录，不将历史计划改写成事后验收结果。下文第 1–7 节中的 `tools/manual_recording/`、拟定 CLI/schema、P1–P6“待实现”和 detached HEAD 均指当时状态，不能作为当前运行说明；7.5 为当日随后完成的环境安装和基础测试。实际工具已放到另一个仓库的 `game-agent-stagesft/Stable-retro/human_data_recording/`，入口为 `sretro-record` / `sretro-validate`，代码 commit 为 `9735c7a1`。本仓库目前使用 `xr-gameagent-record` 分支，package 源码仍以原始 `v1.0.1` 为基线。

历史实现决策（2026-09-05）：Stable Retro 保持原始 `v1.0.1` package，采集程序拟放在包外的 `tools/manual_recording/`，通过现有 API、环境 wrapper 和外部 interface 子类完成需求。此前将 recorder 加入 `stable_retro/`、调整包内 interactive 和增加 package extra 的方案，已统一替换为下文的包外实现计划；2026-09-06 又将工具位置调整为上述独立仓库目录。Stable Retro 首次安装可能需要编译；之后修改采集程序无需重新编译 emulator。

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

## 8. 2026-09-09 已实现：单局 CLI 多次启动追加录制

日期基准：America/New_York。用户希望保留 `--stop-on-done`，每次只录一局、退出进程；反复调用同一命令和输出路径时自动生成 `000000`、`000001`、`000002`，不再因目录存在而失败。

### 8.1 实现范围与使用方式

代码仍位于独立仓库的 `game-agent-stagesft/Stable-retro/human_data_recording/`；录制实现 working note 统一维护在本文件，不放到该仓库的 `context/implementation/`。新增显式 `--append`，第一次目标不存在时创建 session，之后对完整关闭的 session 追加新 episode。不带该开关仍拒绝已有目录，不默认覆盖。无需修改 Stable Retro package 或重新安装 Conda 环境，现有 editable 安装直接生效。本次没有 Git stage/commit/push，没有修改用户录制、ROM 或其他项目代码。

```bash
conda activate stable-retro-recording
session_dir=/scratch/gpfs/CHIJ/xinran/projects/game-envalgm/human-recordings/mario/session

# 每次重复执行这条命令；保持同一个 session_dir，不再每次 mktemp。
sretro-record --game SuperMarioBros-Nes-v0 \
  --record-dir "$session_dir" --record-bk2 --stop-on-done --append

sretro-validate "$session_dir"
```

### 8.2 关键设计

- `session_lifecycle.py`：Linux `flock` 非阻塞独占写锁，完整旧 session 校验、配置兼容检查、legacy run 迁移和累计 writer 统计。`.recording.lock` 文件保留以避免删除重建造成 inode 竞争；关闭/崩溃时 OS 释放锁，锁文件存在不代表仍占用。
- 新 session 和 append 都参与同一锁协议。候选目标需有连续、已登记的 episode 目录；有空洞、未登记目录、损坏数据、error/未关闭 manifest 时拒绝，不自动跳过或修复。
- `state_utils.py` 将资产读取与写入分离：候选 data/scenario/metadata/semantic/Lua 先读入内存计算哈希，只有新 session 才写 assets；追加不会先覆盖旧配置再进行比较。
- 兼容性要求相同 game/integration、ROM/core/native/Python API identity、按钮布局、action mode、semantic map、core info、归档 assets，以及初始 state 名称/字节。初始化哈希对旧 v1 可从第一局 `initial.state` 描述取得。不同 state/game/scenario 使用独立 session。
- `TrajectoryRecorder` 从旧 manifest 接续 episode 编号和 `global_step`；旧 episode 内所有文件保持字节不变。新 episode 自己的 step/frame 从 0 开始。
- 新增 `session.json.runs[]` 保存每次启动的半开 episode/step 区间、状态、实际工具源码/依赖来源、初始化/采集选项、writer 和 performance。顶层来源仍描述首次采集；旧版 live-state session 首次追加时作为 legacy run 0，旧 episode 不改写。
- 顶层 writer 计数累计，queue capacity/peak 取历史最大值；performance 明确只描述最新 run，不能合并不同启动的延迟百分位数。`--max-steps` 仍按本次进程步数限制。
- Validator 验证 run/episode/step 对应关系和计数；`monotonic_ns` 仅在同次启动内保持单调，允许跨机器或系统重启后时钟基准变化。
- GUI 显示接续后的真实 episode 编号。`--stop-on-done` 仍保存 terminal transition/PNG/state、关闭 BK2、排空 writer 后退出，不 reset；不带该开关仍支持单进程多局。
- Esc 正常退出的 interrupted episode 可保留并追加下一局，不续写旧局。只有 frames + initial.state 的更早格式仍可读，但拒绝混合追加 live states。
- 追加校验失败或新 episode 开始前失败，不改写旧 manifest。新局开始后出错，旧文件保留，整个 session 标记 error；不支持中途恢复/崩溃自动修复。错误清理也会先关闭 writer 再释放锁。

### 8.3 验证结果

环境：`/home/xl9353/.conda/envs/stable-retro-recording/bin/python`。隔离测试根：`/tmp/stableretro-append-tests.4OOwYYnY`；测试产物不进入 Git，也不保证持久保存。

1. 轻量单元检查：81 passed、30 deselected（当时尚未补入最后一个原生 legacy 测试）。覆盖自动编号、旧数据不变、配置/语义/资产不兼容、初始 state 内容变化、损坏/孤立 episode、旧格式迁移、跨 run 时钟、错误写盘和独立进程锁。
2. 最终完整回归：**112 passed in 47.03s**，含 Xvfb GUI 和原生 emulator/replay 检查。
3. Airstriker（Genesis）、SuperMarioBros（NES）、SuperMarioWorld（SNES）各使用三个独立 CLI 进程，重复 `--append --stop-on-done`。每款生成 3 个 run / 3 个 episode，旧 episode 和 assets 哈希不变，静态校验、完整 replay、BK2、导出 state 初始化检查全部通过。
4. 上述原生重启测试用显式 synthetic scenario 在第一步触发环境 done，使测试有界：每款 3 transitions、6 PNG、6 live states，另有 3 个 initial.state。不是人类通关测试，也不声称验证自然 game-over 条件。
5. GUI 测试检查追加后 HUD episode=1、只录一局退出、重复 CLI 打开/关闭窗口，以及两局 BK2/replay；全部通过。
6. 旧版 live-state session 的原生兼容测试：在隔离数据中模拟旧 v1 manifest（没有 runs/初始化哈希），追加后旧文件不变，完整 replay/BK2 通过。早期 frame-only 格式仍可静态校验。
7. `python -m human_data_recording.prepare_action_maps --check`：828 ready、0 invalid、0 files_needing_update。没有改动或重生成 828 份 mapping；本次没有重新运行全 828 游戏原生审计。
8. `git diff --check` 无空白错误；CLI `--help` 已显示 `--append`。

最终完整测试命令（在 `game-agent-stagesft/Stable-retro/human_data_recording/` 目录执行；重跑时使用新的临时输出路径）：

```bash
xvfb-run -a -e /tmp/stableretro-append-tests.4OOwYYnY/xvfb-full-v2.log \
  -s '-screen 0 1280x960x24' \
  env RUN_GUI_TESTS=1 PYTHONDONTWRITEBYTECODE=1 \
  /home/xl9353/.conda/envs/stable-retro-recording/bin/python -m pytest \
  -q -p no:cacheprovider \
  --basetemp=/tmp/stableretro-append-tests.4OOwYYnY/full-v2
```

此前首轮存在一个新增测试断言位置错误，已修正；沙箱内 Xvfb 因不允许 bind 本地 X11 socket 导致 GUI 检查失败，诊断日志已确认。最终使用获准的沙箱外短 CPU/Xvfb 测试运行，保留了完整 GUI 检查。

### 8.4 剩余边界

- 人在真实桌面长时间录多局、海量历史数据追加启动耗时仍需实际验收。追加前会完整扫描旧 PNG/state，复杂度随历史数据量增加。
- OS advisory lock 约束遵循协议的工具进程；不防止其他程序直接改文件，不是对不可信共享存储的事务系统。
- 旧版工具不认识跨 run 时间边界；追加后的 session 应使用更新版 validator。
- 不承诺修复此前六款 state 恢复差异或三款 replay 差异；录制追加与原生确定性问题相互独立。

操作说明已同步到工具 README 和 `docs/recording_format.md`。

## 9. 2026-09-09 探索结论：按 world / level 选择初始关卡

本节为只读探索和后续设计建议，不表示新增了 `--world` / `--level` 参数，也没有修改 Stable Retro package 或采集工具实现。当前已经可以使用 `--state` / `--state-file` 从已有 snapshot 初始化；底层调用 `retro.make(state=...)` / `env.load_state()`，随后 `reset()`。Stable Retro 没有通用的 world/level 选关 API，命名 state 才是跨游戏接口。源码依据为 [retro_env.py](../../stable_retro/retro_env.py) 和采集工具的 `state_utils.py`。

### 9.1 本地 Super Mario Bros state 清单与实测

游戏 ID：`SuperMarioBros-Nes-v0`，默认 state 为 `Level1-1`。通过 `retro.data.list_states(game, retro.data.Integrations.STABLE)` 枚举得到 12 个命名 state，对应 9 个不同 world/level 组合：

| World / Level | 已有 state |
| --- | --- |
| 1-1 | `Level1-1`；变体 `Level1-1-99lives` |
| 1-4 | `Level1-4` |
| 2-1 | `Level2-1`；变体 `Level2-1-clouds`、`Level2-1-clouds-easy` |
| 3-1 | `Level3-1` |
| 4-1 | `Level4-1` |
| 5-1 | `Level5-1` |
| 6-1 | `Level6-1` |
| 7-1 | `Level7-1` |
| 8-1 | `Level8-1` |

当前没有 `Level1-2`、`Level2-2` 等现成文件，并未覆盖所有 world/level 组合。

对全部 12 个 state 分别执行：创建原生 env、检查 state 被接受、reset、5 步 NOOP、再次 reset。12 个都通过；返回 RGB shape 均为 `[224, 240, 3]`，短步进未触发 done，每个 state 两次 reset 的 RGB 哈希一致。另读取 `env.data.lookup_all()` 和首步 `info`，确认 `levelHi/levelLo` 与上述名称对应，例如 `Level2-1` 为 `1/0`。本地 v1.0.1 的 `reset()` 返回空 info，初始变量校验不能直接依赖 reset info 中存在这两个字段。

本次只做内存中的初始化/短步进检查，没有创建新的人类录制，也没有对所有 12 个 state 执行完整录制/replay 或长时通关验收。

### 9.2 不改代码即可使用的命令

```bash
conda activate stable-retro-recording
sretro-record --game SuperMarioBros-Nes-v0 --state Level2-1 \
  --record-dir /scratch/gpfs/CHIJ/xinran/projects/game-envalgm/human-recordings/mario/world2-level1/session \
  --record-bk2 --stop-on-done --append
```

重复同一条命令时，每次从该 snapshot 开始新局、接续 episode 编号。当前 `--append` 要求同一 session 固定初始 state，因此切换关卡应使用独立 session：

```text
mario/
  world1-level1/session/episodes/000000、000001、...
  world2-level1/session/episodes/000000、000001、...
  world3-level1/session/episodes/000000、000001、...
```

若后续确实需要在同一 session 混合不同初始关卡，需要另行调整采集工具的初始化兼容规则及校验；每局虽然已经保存独立 initial.state，当前追加 guard 仍会拒绝切换 state。本节未放宽这一限制。

### 9.3 Snapshot 内容与 episode 结束条件

State 保存的是完整模拟器状态，不只是 world/level，也包含位置、生命、时间、分数、敌人等。不能仅凭名称将它视为统一条件的“全新关卡开始”。本次 reset 后实测：

| State | 剩余时间 time | score |
| --- | ---: | ---: |
| `Level1-1` | 400 | 0 |
| `Level1-4` | 252 | 4990 |
| `Level2-1-clouds` | 237 | 605 |

`Level1-1-99lives` 是不同生命数的变体；`clouds` / `clouds-easy` 也不是普通 `Level2-1` 的等价初始化。正式采集要验收各 snapshot 的画面、位置、生命和时间等条件，而不是只看文件名。

对没有现成 snapshot 的关卡，可先游玩到目标位置，选择实际录制保存的对应 `.state`，通过现有 `--state-file` 初始化；不需要额外 ROM。重用文件仍需相同 ROM/core 的兼容性验证。仅修改 RAM 的 world/level 数字不能保证地图和内部运行状态一致，不作为默认选关方案。

初始化关卡与 episode 结束条件相互独立。当前 Mario 默认 scenario 的 done 条件是 `lives == -1`，`--stop-on-done` 不会在通过 2-1 时自动停止。如果目标变为“每条 demonstration 只覆盖一个关卡”，需要另行定义关卡完成/失败标准和相应 scenario。加载录制 snapshot 后 `reset()` 仍会多推进一帧 NOOP，并重新开始 reward bookkeeping。

### 9.4 后续可选参数设计与验证计划（尚未实现）

建议保留 `--state` 作为通用接口；`--world 2 --level 1` 只是 Super Mario Bros 特定的便捷 selector，通过显式映射解析到 `Level2-1`：

- world 和 level 成对提供，与 `--state` / `--state-file` 互斥。
- 只接受真实存在、经过核验的 state；缺失关卡明确报错并列出可选项，不静默回退默认关卡。
- 普通 2-1 默认选择 `Level2-1`，不擅自切换到 clouds / easy / 加命变体。
- 记录请求的 world/level、实际解析的 state 名称、snapshot 哈希和核验后的实际关卡。不同游戏不复用未经验证的 RAM 字段解释。
- 不强行统一所有游戏的关卡命名：Mario 3 当前有 `1Player.World1.Level1` 等名称；Mario World 使用 `YoshiIsland1`、`DonutPlains1` 等。
- 如果未来实现，测试应覆盖有效/缺失/冲突参数、变体选择、实际关卡变量和初始画面、对应录制/replay；追加模式需验证同 state 接续成功、不同 state 按既定策略拒绝。跨关卡单局终止标准需独立验收。

当前建议：先用现有 `--state`，按初始关卡分 session；明确初始条件与终止标准后，再决定是否添加 world/level selector。

## 10. 2026-09-09 实现：沿用 `--state`，记录实际初始化关卡

### 10.1 用户选择与范围

用户决定直接使用 `--state`，并在 recording log 中保留 `world1-level1` 这样的初始化信息。此次只修改包外工具 `game-agent-stagesft/Stable-retro/human_data_recording/`，不修改 Stable Retro v1.0.1 package、ROM 或 integration。不新增 `--world` / `--level`；已有 `--state-file` 和默认 state 路径保留。

`--record-dir` 的含义不变，仍由调用者选择完整 session 根路径；不会自动拼接 world/level 子目录。推荐手动按初始关卡组织目录，重复使用同一个 state + session 路径录多局；改变 state 名称或内容必须另开 session。

```bash
conda activate stable-retro-recording
session_dir=/scratch/gpfs/CHIJ/xinran/projects/game-envalgm/human-recordings/mario/world2-level1/session

# 在 desktop/X11/VNC 环境运行；每次执行追加一局。
sretro-record --game SuperMarioBros-Nes-v0 --state Level2-1 \
  --record-dir "$session_dir" --append --stop-on-done --record-bk2

sretro-validate "$session_dir"
sretro-validate "$session_dir" --replay --check-bk2
```

本机采集工具已 editable 安装，在同一个 `stable-retro-recording` 环境可从任意 cwd 使用，无需重新安装 Stable Retro 或编译 core。上述目录是建议命名，不是本次自动创建的数据目录。

### 10.2 代码与日志结构

实现文件：

- 新增 `src/human_data_recording/initialization.py`：显式 game adapter、post-reset 变量读取和纯函数标签解析；不根据目录或 state 文件名猜测关卡。
- `recorder.py`：每次真正的 `env.reset()` 返回后，在首个 action 前采集 initialization location；写入本局 `episode.json`，并将各 session/run 第一局的初始化信息写入相应 manifest。所有模拟器调用仍在主线程，不增加 reset、step 或 get_state 调用。
- `session_lifecycle.py`：append 不把派生的 `location` 当作启动前兼容性字段；继续严格核对初始 state kind/name/raw SHA-256。旧日志没有 location 时仍可追加，旧 episode 不改写、不补造关卡标签。
- `validate_recording.py`：静态校验初始化 state hash、episode/run 名称与标签关联、标签与保存的变量是否一致；replay 在实际 reset 后重新读取变量，并与录制初始化信息比较。
- `manual_play.py`：沿用原有 `--state` 参数，仅更新 help。工具 `README.md` 和 `docs/recording_format.md` 更新使用方式与 schema；不在工具仓库新增 implementation working note。

记录位置：`session.json.initialization`、`session.json.runs[i].initialization`、`episodes/NNNNNN/episode.json.initialization`。原有 kind/name/raw SHA-256 保留，新增 `location`；例如 World 2-1：

```json
{
  "kind": "integration_state",
  "name": "Level2-1.state",
  "raw_sha256": "<pre-reset initial.state 的解压后 SHA-256>",
  "location": {
    "phase": "post_reset",
    "status": "resolved",
    "label": "world2-level1",
    "world": 2,
    "level": 1,
    "source": "integration_variables:SuperMarioBros-Nes-v0:v1",
    "variables": {"levelHi": 1, "levelLo": 0, "lives": 2, "time": 400},
    "reason": null
  }
}
```

例子只展示部分 variables；实际保留该 Mario integration 的全部 `env.data.lookup_all()` 结果，包括 coins、score、scrolling、xscroll 等。Stable Retro 的 reset info 仍原样保存为 `{}`，不会以初始化变量替换它。

时间边界必须区分：initialization raw SHA-256 标识 **reset 输入** `initial.state`；location 描述 **reset 返回后** 的 frame[0]/state[0]。Stable Retro 本身 reset 会推进一帧 NOOP，本改动不额外推进。location 是初始关卡标签，不随游玩跨关而改变；逐步的实际游戏变量仍在 transition.info 中。

`Level1-1` 和 `Level1-1-99lives` 都可能标记 `world1-level1`，但完整名称/hash 及初始 lives/time 不同，不能当作同一种 snapshot。外部 state 即使叫 `00000001.state`，Mario adapter 仍从实际变量确定关卡；可能是关卡中途状态，不代表干净的关卡开局。

顶层 initialization 描述整个 session 第一局，各 run 描述该次调用第一局；每局完整上下文以自己的 `episode.json` 为准。transition 不重复写同样的初始化 metadata，通过 episode_id 与 episode.json 关联即可构造带初始关卡标签的数据样本。

### 10.3 已验证与未知的边界

目前只对 `SuperMarioBros-Nes-v0` 启用已核实规则：`world=levelHi+1`、`level=levelLo+1`，有效范围为 1–8 / 1–4。不将规则套到 Mario 3、Super Mario World 或其他游戏。其他游戏仍有准确 state 名称/hash，但 location 的 label/world/level/source 为 null，status=unavailable、reason=unsupported_game、variables={}。Mario 缺少变量或值越界时保留原变量，并标记 missing_or_invalid_level_variables，不回退猜测 World 1-1。

本地 12 个 Mario 命名 state（9 个不同 world/level）全部覆盖此次有界录制测试；名单见 9.1。仍不是任意 world/level 都有现成 snapshot；缺少 Level1-2 等关卡时需要另行准备兼容 state。本次没有修改 scenario 或 episode 结束定义，仍以原有 terminated/truncated 为准。

### 10.4 验证计划与本次结果

采用单元校验 → 真实 Mario 快照专项 → 全套 CPU/Xvfb 回归；数据全部写新建 `/tmp` 目录，不覆盖旧录制。环境 Python：`/home/xl9353/.conda/envs/stable-retro-recording/bin/python`。

- 单元回归：`100 passed, 46 deselected in 2.99s`。包含已知/未知游戏解析、缺失/无效变量、初始化日志损坏检查，以及原有录制/append 测试。
- Mario 专项：`15 passed, 19 deselected in 5.95s`。全部 12 个命名 state 各实际 reset + 3 个 NOOP transition、保存 live PNG/state/BK2，再静态校验和 replay/BK2 对齐；另覆盖数字文件名 custom state、连续三次 CLI append 与换 world 拒绝、伪造一致标签后 replay 必须拒绝。
- 完整回归：`146 passed in 54.78s`，包含真实 NES/SNES/Genesis 录制、GUI/Xvfb、stop-on-done、append、旧日志兼容、replay、BK2 和 state initialization 原有测试。未跳过 GUI；Xvfb 需要本地 X11 socket 权限，以批准的沙箱外短 CPU 命令运行。
- 两个 repo 的 `git diff --check` 通过。尚未 commit/push，保留原有 append 改动和其他无关工作。

验证产物根：`/tmp/stableretro-initialization-tests.HFlRSj/`，子目录 `unit/`、`mario/`、`full/`；Xvfb 日志 `xvfb.log`。专项 World 2-1 的示例日志：`mario/test_all_local_mario_states_re3/recording/episodes/000000/episode.json`。

完整回归命令（重跑需替换 basetemp 为未使用的新目录）：

```bash
xvfb-run -a -e /tmp/stableretro-initialization-tests.HFlRSj/xvfb.log \
  -s '-screen 0 1280x960x24' \
  env RUN_GUI_TESTS=1 PYTHONDONTWRITEBYTECODE=1 \
  /home/xl9353/.conda/envs/stable-retro-recording/bin/python \
  -m pytest -q -p no:cacheprovider \
  --basetemp=/tmp/stableretro-initialization-tests.HFlRSj/full
```

验收限制：12-state 专项是短时程序输入，不是人工打通 12 个关卡；Xvfb 回归不代替真实 desktop 长时人工验收。本次没有重跑 828 游戏全量 native audit，也未声称其他游戏已有 world/level adapter。静态校验只能检查已记录证据的内部一致性；与真实 snapshot 是否一致由 replay 进一步检查。已有日志不自动升级或补标签。

## 11. 2026-09-09：多-state 游戏名称资产与可重复检查

### 11.1 全量检查结果与收录范围

按用户要求，在采集工具 `assets/` 中按游戏存放多个预置 state 的名称；不复制 `.state` 文件或 ROM，不修改 Stable Retro integration。遍历安装中的 `Integrations.ALL`（stable/experimental/contrib；当前没有注册 custom 路径），以 `retro.data.list_states()` 的公开结果为准：

| 范围 | 游戏数 | 公开 state 数 | 多-state 游戏数 |
| --- | ---: | ---: | ---: |
| 全部 integrations | 1,033 | 1,807 | 68 |
| ROM ready | 828 | 1,573 | 57 |
| 本次按游戏生成的目录 | 68 | 842 | 68 |
| 目录内 ROM ready 子集 | 57 | 802 | 57 |

全部 1,033 个游戏都有至少一个公开 state；未导入 ROM 的 205 个游戏不能因为有 state 就被视为可运行。828 个 ready 游戏的 ROM 哈希重新验证通过，无 invalid ROM；其中 771 个只有一个公开 state，因此本次不生成单独文件，但它们仍然支持 `--state`。

目录覆盖全部 68 个多-state 游戏，包含 stable 59、experimental 4、contrib 5；11 个尚无 ROM 的游戏也保留清单，明确标记 `rom_ready_at_scan=false`。不把这些游戏计入当前 ready 集合。

先前全量只读检查对 1,807 个公开 state 及 14 个下划线开头的内部 state 执行 gzip EOF/CRC、解压非空检查，全部通过；内部 state 不放入本次公开目录。生成器重新检查目录内 842 个 state。完整只读报告曾输出于 `/tmp/stableretro-preloaded-states-audit.UcjTpN/inventory.json`；新的 source assets 不依赖该临时报告，直接扫描 live integrations。

### 11.2 文件结构与字段

工具根：`/scratch/gpfs/CHIJ/xinran/projects/game-envalgm/game-agent-stagesft/Stable-retro/human_data_recording`。

```text
src/human_data_recording/
  prepare_state_catalog.py
  assets/
    state_catalog.json                       # 总索引，游戏文件哈希和汇总
    preloaded_states/
      README.md                              # 字段、范围和使用说明
      SuperMarioBros-Nes-v0.json              # 12 个公开 state
      SuperMarioBros3-Nes-v0.json             # 6 个
      SuperMarioWorld-Snes-v0.json            # 25 个
      SonicTheHedgehog-Genesis-v0.json        # 17 个
      ...                                    # 共 68 个游戏 JSON
tests/test_state_catalog.py
```

每游戏 JSON：game、integration、system、`state_count`、`rom_ready_at_scan`、`rom_sha1`、`states`、单玩家默认选择、原 metadata 默认配置、`issues`。其中：

- `states[].name` 是直接传给 `--state` 的名称，例如 `Level1-1`。
- `states[].filename` 是实际文件名，例如 `Level1-1.state`；保留精确大小写。
- `states[].integration` 记录 state 所在的 integration 类别。
- `states[].raw_sha256/raw_size/gzip_integrity` 是解压后摘要、字节数和 gzip 完整性结果，不代表 native 接受该 state 或 replay 已通过。
- `single_player_default` 按 `RetroEnv(players=1)` 的优先级选择：先取非空 `default_player_state[0]`，否则取 `default_state`；记录 name/filename/source/status，不自行修正拼写。
- `metadata_default_state` 与 `metadata_default_player_states` 保留原始配置，用于核对默认优先级和已知异常。

索引 `state_catalog.json.games[game]` 提供相对 JSON 路径、文件 SHA-256、state 数、ROM-ready 标记和问题列表；summary 记录全部扫描范围与目录范围的不同计数。生成结果确定，不嵌入机器绝对路径或变化的生成时间。ROM readiness 是生成时的本地安装快照，不保证新机器环境，日期依据本记录及 Git 历史。

默认引用异常保留为 `single_player_default_missing_file`：`NHL94-Genesis-v0` 和 `NHL941on1-Genesis-v0` 都配置了末尾多句点的 `PenguinsVsSenators.start.`；前者 ROM ready，后者缺 ROM。正确的 `PenguinsVsSenators.start` 仍列在各自 states 中，没有修改原 metadata。

### 11.3 查询、刷新与采集行为

```bash
conda activate stable-retro-recording

# 只读：重新扫描并检查 source assets 是否一致；不同则 exit 1。
python -m human_data_recording.prepare_state_catalog --check

# 明确刷新：只更新每游戏 JSON 和汇总 JSON，不复制/删除 ROM 或 state。
python -m human_data_recording.prepare_state_catalog --write
```

不加参数只预览统计，不写文件。`--assets-dir` 可指定独立输出目录。生成器在所有扫描/规划完成后才写 JSON，重复生成内容不变时不会重写；发现目录中的多余/过期 JSON 会报错要求人工复核，不自动删除。生成 JSON 不承载手写语义标签，刷新可以替换其元数据内容。

查询一个游戏的名称，不依赖当前工作目录：

```python
import json
from importlib.resources import files

asset = files("human_data_recording").joinpath(
    "assets", "preloaded_states", "SuperMarioWorld-Snes-v0.json"
)
game = json.loads(asset.read_text())
print("\n".join(state["name"] for state in game["states"]))
```

采集 CLI 不以目录作为允许列表，也不会自动选 state；仍直接使用 Stable Retro 的已有 integration。用户选择 `states[].name` 作为 `--state`，experimental/contrib 游戏使用相应 `--integrations`。原有 initialization 记录规则、append 固定初始 snapshot 约束、world/level adapter 和 stop-on-done 语义均不改变。

### 11.4 验证、打包与限制

- 新增目录专项：`16 passed in 12.86s`。验证默认优先级、错误/空 gzip、确定性、只读 preview/check、write 幂等、多余文件不删除、安全文件名，以及全部 source JSON 与 live integrations/哈希一致性。
- 非 GUI 回归：`149 passed, 13 deselected in 71.23s`，包括现有录制、append、初始化日志、真实 emulator replay/BK2 和 action catalog 测试。本次没有修改 UI 或录制流程，因此未重复运行 GUI；此前第 10 节的 146 项完整回归包含 GUI。
- `python -m human_data_recording.prepare_state_catalog --check`：69 份 JSON 全部一致，`files_needing_update=0`。
- 与前一轮独立全量 inventory 交叉比较：68 个游戏的 state 名称、state hash、ROM-ready 状态及默认引用结果全部一致。
- 在 `/tmp` 的源码副本构建 wheel，确认包含 68 个每游戏 JSON、总索引和 README，内容逐字节与 source assets 一致；不含 ROM、`.state`、BK2、PNG 或 JSONL。没有重新安装环境，也没有在工具 repo 留下本次 build 产物。
- `pyproject.toml` 补充 package-data；`.gitignore` 只新增这两个 JSON 路径的例外，避免父 repo 的全局 JSON ignore 隐藏 source assets。已用 `git check-ignore -v` 核对例外生效，`git diff --check` 通过。尚未 stage/commit/push。

验证临时目录：`/tmp/stableretro-state-catalog-tests.THs3rz/`（unit、non-gui、package、wheels）。wheel SHA-256：`cde1fc32d9a291a31af78665c71c3d816bfc54df65b0104ecc0d4a442a2c92c1`。

边界：名称目录不是 state 二进制备份、world/level semantic mapping 或每个快照的运行认证。某些名称表示游戏中途、多人比赛、角色或生命数变化，不保证适合单玩家 demonstration；正式使用仍需对所选 state 验证实际初始化/录制/replay。本次没有执行 842 个 state 的 native 全量测试，没有更改 ROM、integration、场景或旧录制数据；implementation working note 只补充于本文件。

## 12. 2026-09-09 Git 交付记录

用户要求分别提交/推送工具目录与本文件。采集工具已在 `game-agent-stagesft` 的 `xr-sft` 分支创建本地 commit `70f324f2a5c67a906e571ad795ff52414f0fff8a`：`feat(recording): append episodes and catalog initialization states`。仅包含 `Stable-retro/human_data_recording/` 内已校验的 90 个文件（其中 68 个游戏 state JSON + 1 个索引）；没有加入 ROM、state 二进制、录制产物、其他工程修改或未跟踪文件。

本次提交前 `prepare_state_catalog --check` 再次通过（0 个待更新 JSON），staged diff 空白检查及范围检查通过。第 10/11 节的“尚未 commit/push”描述各实现验证结束时的历史状态，以本节的后续交付记录为准。

在本记录提交准备时，采集工具尚未推送：自动安全审核要求用户明确确认 remote `https://github.com/Chengshuai-Shi/game-agent.git` 的 `xr-sft` 分支；此外 HTTPS fetch 无可用凭据，已有 SSH 认证返回 publickey denied。没有更改 remote/认证配置，也没有使用 force push。本文件在当前 `Stable-Retro` repo 的 `xr-gameagent-record` 分支单独提交；最终远程同步状态应以实际 push 结果及 remote ref 为准，不能将本地 commit 当作已上传。
