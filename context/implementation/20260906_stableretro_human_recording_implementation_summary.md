# 2026-09-06：手动采集工具实现与验证总结

状态：采集工具已实现并完成自动化验证；新录制直接保存实际游玩轨迹的 frame、native state、action、reward 和 episode 边界，不依赖事后 replay 才获得 state。本文更新 [2026-09-05 working note](20260905_stableretro_human_manual_play_recording_working_note.md) 的实施状态；旧文保留历史方案，本文件记录实际落地结果。

## 1. 仓库边界、版本和环境

- Stable Retro 源码：`/scratch/gpfs/CHIJ/xinran/projects/game-envalgm/Stable-Retro`；远端 `xinranliang/Stable-Retro`，分支 `xr-gameagent-record`。基线为 tag `v1.0.1` / `ec7a62718a1f99f34bf5e5d5c57255c9a53df507`，此前环境/ROM 文档提交为 `83c65971`。这次仅更新 `context/`，不修改上游 package 或重新编译。
- 实际工具：`/scratch/gpfs/CHIJ/xinran/projects/game-envalgm/game-agent-stagesft/Stable-retro/human_data_recording`；属于 `Chengshuai-Shi/game-agent` 仓库的 `xr-sft` 分支，不是本仓库的 `tools/manual_recording/`，也不是 nested Git repo。
- 工具代码提交：`9735c7a12b0a89f9c183d4b27187d92e9e4c9a76`，`Add Stable Retro human recording with live emulator states`。这是已核对的本地提交标识，不以此单独宣称远端 push 已成功；两仓库的提交和推送分别管理。
- 专用环境：`stable-retro-recording`，Python 路径 `/home/xl9353/.conda/envs/stable-retro-recording/bin/python`；Stable Retro 和工具均 editable 安装。Python 3.12.14、Stable Retro 1.0.1、NumPy 1.26.4、Gymnasium 1.3.0、Pyglet 1.5.31、Pillow 12.3.0；`pip check` 通过。
- 未修改或重新验收 `game-verl` / `game-verl-opensrc`。不需要 Torch、VERL、CUDA；共享 editable 源码不能随意移动、删掉或换 tag 后不验证就继续用。

工具自己的文档是运行和字段定义的依据。以下链接适用于当前两个仓库同级存放的本地工作区；在远端阅读时，应到工具仓库上述 commit 的 `Stable-retro/human_data_recording/` 查阅相应文件：

- [README：安装与操作](../../../game-agent-stagesft/Stable-retro/human_data_recording/README.md)
- [安装说明](../../../game-agent-stagesft/Stable-retro/human_data_recording/docs/installation.md)
- [实际 recording format v1](../../../game-agent-stagesft/Stable-retro/human_data_recording/docs/recording_format.md)
- [完整实现与验证记录](../../../game-agent-stagesft/Stable-retro/human_data_recording/docs/20260906_validation.md)

实际模块按职责拆分为 `manual_play`、`interface`、`action_mapping`、`recorder`、`writer`、`schema`、`state_utils` 和 `validate_recording`。`ManualInterface` 继承原有 `Interactive`，`RecordingEnv` 包装原生环境；使用 `em.get_state()`、`em.set_state()`、`initial_state`、`record_movie()` / `stop_record()` 等现成 API，不 monkey-patch 包源码。

## 2. 当前启动和录制方式

已有环境无需重装；editable 工具可以从任意工作目录启动。仅首次安装工具时，进入工具目录执行 `python -m pip install --no-build-isolation --no-deps -e .`，前提是所需依赖已经按安装说明准备好。ROM 已完成批量导入，覆盖和限制见 [ROM 验证记录](20260905_rom_validation.md)；不把 ROM 复制到 session 或 Git。

桌面验证示例（要求有效的 X11 forwarding / VNC；这里启动的是 Mario）：

```bash
conda activate stable-retro-recording

# mktemp 的模板末尾必须有足够的 X；工具要求 session 子目录尚不存在。
recording_check_dir=$(mktemp -d /tmp/stableretro-manual-check.XXXXXXXX)
printf 'Recording output: %s\n' "$recording_check_dir/session"
sretro-record --game SuperMarioBros-Nes-v0 \
  --record-dir "$recording_check_dir/session" --record-bk2 --stop-on-done

# 录制正常退出后，在同一个 shell 校验；不需要 replay。
sretro-validate "$recording_check_dir/session"
```

`--game` 必填；不指定 `--state` 使用游戏默认 snapshot。也可改为 `--game Airstriker-Genesis-v0 --state Level1`。方向键移动；Z/X/C 对应 A/B/C，Mario 的 Z 跳跃、X 对应 B，Airstriker 的 X 开火。P 暂停/恢复，失焦清空按键并暂停，Esc 退出；HUD 不会写入 PNG。没有 `--record-dir` 时只玩不录，BK2 也只有显式 `--record-bk2` 才保存。

`--stop-on-done` 默认关闭；开启后只录一个环境 episode：

```text
env.step() 返回 terminated/truncated
  -> 捕获 terminal PNG/state，提交最后一条 transition
  -> 关闭 BK2、排空 writer、完成 episode/session
  -> 关闭窗口，不再 reset
```

终止以 integration 的 scenario 为准，不按掉一条命或画面文字猜测。Airstriker 默认需同时 `gameover == 1` 和 `lives == 0`；其他游戏须核对自己的 scenario。单 episode 模式禁用 F9；Esc、窗口关闭、Ctrl-C、`--max-steps` 仍可提前结束，未达到 done 的 episode 标记 `interrupted`，不伪造终止。未开启开关时，done 自动 reset，F9 可手动 reset，各 episode 分目录编号。

Headless 使用 `--headless --max-steps N --keys RIGHT,Z` 等固定输入，只用于有界诊断，不是人类 demonstration。`/tmp` 不保证持久保存；结束后应将整个 session（含 session.json/assets）移到持久存储，不要只保留一个 episode 子目录用于完整验证。

## 3. 录制时保存 state 与 sample 对齐

单玩家、`frame_skip=1`；每条 transition 对应一次原始 `env.step()`，无按键时也保存 NOOP，暂停时不产生 transition。

```text
SESSION/
  session.json
  assets/                       # 配置、语义及可归档 Lua，不含 ROM
  episodes/000000/
    episode.json
    initial.state               # reset 输入：pre_reset_input
    frames/00000000.png ...      # frame[0..N]，共 N+1 张
    states/00000000.state ...    # 实际录制的 state[0..N]，共 N+1 个
    states.jsonl                # source=recording；逐帧关联及哈希
    transitions.jsonl           # N 条动作 transition
    recording.bk2               # 可选，每 episode 独立关闭
```

`reset()` 返回后立即保存 state[0]；每次 `step()` 返回后、任何后续 step/reset 之前，在主线程读取 `bytes(env.em.get_state())`，得到 state[t+1]。终止 state 也保存。另一个 `initial.state` 是 reset 的输入而不是 post-reset state[0]，所以一个 N 步 episode 总计 N+2 个 `.state` 文件；0 步退出也有初始图片、post-reset state 和 initial.state。

真实字段名以工具格式文档为准，不沿用旧计划中的 `obs_frame` / `effective_action` / `semantic_action`：

| sample 内容 | `transitions.jsonl` 字段 |
| --- | --- |
| 当前 / 下一张图片 | `frame` / `next_frame` |
| 当前 / 下一 emulator state | `state` / `next_state` |
| 图片原始像素哈希 | `frame_sha256` / `next_frame_sha256` |
| 解压后的 native state 哈希 | `state_sha256` / `next_state_sha256` |
| 请求 / 实际按钮向量 | `requested_action` / `action` |
| 动作 index、按钮和语义 | `action_index`、`buttons_pressed`、`semantic_actions` |
| 反馈和边界 | `reward`、`episode_return`、`terminated`、`truncated`、`info` |
| 定位 | `episode_id`、`step`、`global_step`；输入来源在 `input.source` |

路径相对当前 episode。`action_index` 是实际按钮向量的 bitmask，不是 `Actions.DISCRETE` index；可用 `ActionCodec(session["buttons"]).from_index(index)` 恢复向量。语义未配置时回退 `button:NAME`，不猜测游戏行为；语义描述输入指令，不代表动作成功。可直接构造 `(image, state, action, reward, next_image, next_state, done)`，其中 done 为 `terminated or truncated`，同时保留两个原始标志。

`session.json` 新增 `state_capture=recording` 和 gzip level 1 描述；`states.jsonl` 关联 frame_id、phase、解压后 raw SHA-256/size 和 RGB/RAM 哈希。录制 state 不带 `trajectory_verified`，保存成功不等于已重新加载验证。episode.states=N+1；writer.states 还计入每个 episode 的 initial.state。

frame/state 在入队前复制或固定为不可变字节，后台只做 PNG 编码、gzip 压缩和写盘，不调用 emulator。一个有界队列 slot 放一对快照；先分别原子发布 PNG 和 state，再 append 引用它们的索引/transition。它不是跨文件事务，写入失败可能留下未引用文件，但不能被认证为完整 session。队列满时降低运行速度，不丢帧/state；`--queue-size` 调整待写数量，不能保证所有游戏/存储都达到实时 FPS。

## 4. 静态校验、可选 replay 和 custom initialization

`sretro-validate SESSION` 不执行 replay，检查图片、gzip state、原始哈希/大小、文件计数、编号、动作编码/语义、累计 reward 和引用链。旧版只有 frames + initial.state 的 session 仍兼容，但无法补回当时未采集的逐步实际 state。

以下命令均沿用上一节同一 shell 的 `recording_check_dir`：

```bash
# 可选：从归档的 reset 输入和配置重放，精确比较 RGB/RAM/reward/done/info。
# --check-bk2 只用于录制时开启了 BK2 的 session。
sretro-validate "$recording_check_dir/session" --replay --check-bk2 \
  --replay-output "$recording_check_dir/replay"

# 直接使用录制 state 开始新 session，无须先 replay；要求原记录已有 frame 60。
sretro-record --game SuperMarioBros-Nes-v0 \
  --state-file "$recording_check_dir/session/episodes/000000/states/00000060.state" \
  --record-dir "$recording_check_dir/custom-session" --stop-on-done
```

Replay 输出须是新的目录并位于原 session 之外；不覆盖原始数据，出现首个差异即失败。`--dump-states` 仍可在单独 replay 目录额外导出 `source=replay`、`trajectory_verified=true` 的 state，不是新录制获取 state 的必要步骤，也不要求其重新序列化后的字节哈希与 live state 相同。

当前 CLI 的 `--check-state-initialization` 必须和 `--replay --dump-states --replay-output NEW_DIR` 配合，检查的是 replay 导出文件，默认每 episode 最多抽查 3 个非 terminal state；`--state-check-samples 0` 才检查这些输出的全部非 terminal state。真实录制 state 的直接恢复已另在自动化测试中独立验证，不能把这两种检查的来源混为一谈。

两种恢复有不同语义：

```text
原轨迹下一步：set_state(state[t]) -> 原 action[t] -> 核验原 frame[t+1]/RAM
新 episode：  initial_state=state[t] -> reset（额外一帧 NOOP） -> 新 frame[0]
```

`--state-file` 使用后一种；不能期待新初始图片等于原 frame[t]，也不一定等于原 frame[t+1]。Native state 不包含全部 Python wrapper / GameData / Lua reward 历史，新 episode 重新建立 reward bookkeeping。不要将 native snapshot 当成完整训练环境 checkpoint；只加载可信且 ROM/core 兼容的 state、scenario 和 BK2。

## 5. 已完成的验证与剩余工作

实现阶段的结果：57 项非 GUI 测试通过，Xvfb 单独运行 11 项 GUI 测试通过。工具代码提交前又合并运行全套：**68 passed in 34.66s**，产物根为 `/tmp/sretro-git-preflight.WG2swlOC/tests`。本次仅同步 context 文档，不将上述历史测试记为本次重新运行。

- 44 单元测试、13 native/CLI 测试、11 GUI 测试，覆盖动作编码、可变 buffer 深复制、N+1 state、终止/提前退出、慢 writer/磁盘失败、损坏文件和旧格式兼容等。
- 实际游戏覆盖 Airstriker (Genesis)、SuperMarioBros (NES)、SuperMarioWorld (SNES)、SonicTheHedgehog (Genesis, contest Lua scenario)。首次 replay 前直接加载各游戏的 live state，核验 custom reset 及原动作的下一步 RGB/RAM；另验证新 session 的 replay/BK2。
- 600 步 Airstriker：原始环境与开启 recorder + BK2 + live states 的 RGB/RAM/reward/done/info 全部一致。该次 `/tmp`、queue=2 下原始循环 0.454 s、录制循环 3.489 s，目录约 31 MiB；不是 GPFS 或所有游戏实时性能的保证。
- 使用用户当时提供的 `Stable-Retro/000000` 只读数据，合成键盘输入复现 1501 步；三次掉命均继续，step 1500 game over 后停止，仅一个 episode，return=60。任何 replay 前已保存 1502 张 PNG、1502 个 live states（另有 initial.state）；抽查 live state[0]/[396]/[1245] 恢复通过，后续 replay/BK2 通过。原始文件在测试前后 SHA-256 相同。这不是新增人类 demonstration，也不承诺该历史输入目录现在仍存在。
- 已安装 CLI 的 Mario 120 步录制得到 121 个 live states；在 replay 前直接用 live state[60] 新建 30 步 session，两者随后 replay/BK2 通过。max_steps 结束正确标记 interrupted，不冒充 game over。

1501 步诊断摘要曾写于 `/tmp/sretro-live-state.BRXqiWKJ/existing-episode-verification.json`；详细测试记录见工具文档。所有 `/tmp` 路径仅作为当时的溯源记录，不保证长期存在；本仓库不提交这些测试产物。

需要复跑时激活专用环境，使用新的临时目录，避免 pytest 清理已有测试数据：

```bash
recording_test_dir=$(mktemp -d /tmp/stableretro-recording-tests.XXXXXXXX)
xvfb-run -a -s '-screen 0 1280x960x24' \
  env RUN_GUI_TESTS=1 PYTHONDONTWRITEBYTECODE=1 \
  python -m pytest -q -p no:cacheprovider \
  /scratch/gpfs/CHIJ/xinran/projects/game-envalgm/game-agent-stagesft/Stable-retro/human_data_recording/tests \
  --basetemp="$recording_test_dir/tests"
```

尚未完成：真实桌面长时间键盘手感、失焦/暂停和自然 game-over 的完整人工验收；不同存储上的长期性能；`game-verl` / `game-verl-opensrc` 兼容性复测；其他所有已导入游戏的运行与 state 恢复认证。单个游戏、短时 Xvfb 或 68 项测试通过不扩大为这些保证。

## 6. 本次 context 更新的 Git 范围

本次只更新旧 working note 顶部状态和本实现总结，提交到 Stable Retro 仓库的 `xr-gameagent-record`，目标远端为 `https://github.com/xinranliang/Stable-Retro.git`；不用 force push，不向 Farama upstream 推送。工具源码版本由另一个仓库的 `9735c7a1` 标识，不复制代码来制造第二份实现。ROM、图片、JSONL、逐步 state、BK2、环境及测试产物均不纳入此文档提交。
