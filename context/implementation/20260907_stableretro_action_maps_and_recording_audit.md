# 2026-09-07：全部 ROM-ready 游戏的 action mapping 与录制审计

本记录补充 [2026-09-06 实现总结](20260906_stableretro_human_recording_implementation_summary.md)。代码仍在独立工具 `game-agent-stagesft/Stable-retro/human_data_recording/`；不修改本仓库的 Stable Retro package、ROM 或 integration。

工具实现提交（`game-agent` / `xr-sft`）：`de50eef9fd8e078fb9ddc956f85c449400752fe5`（`Add ROM-ready action maps and full recording audit`）。

## ROM 和 mapping 覆盖

重新对已导入 ROM 执行哈希检查：828 个不同游戏 ID 全部通过，其中 NES 301、SNES 188、Genesis 339；stable 815、experimental 13。数字是可匹配 integration 的已安装游戏 ID 数，不是原始 ZIP 文件数，也不是所有游戏都已经人工试玩。

此前 `semantic_maps/` 只有 Airstriker，是因为只有它配置了部分已知的移动/开火等功能标签。其他游戏一直能通过 `env.buttons` 编码 action 并回退为 `button:NAME`，并非缺少输入支持。

现在已生成每游戏一个 mapping JSON，共 828 份。827 个是完整按钮级映射；Airstriker 保留原 6 个标签并补齐其余按钮的回退值。**按钮覆盖不等于功能语义验证**：未查证按钮仍标为 `button:A` 等，不根据 ROM 名称、类型或短期画面/reward 变化猜 jump/fire。仅做录制测试无法自动得出这些游戏特有语义。

工具新增 `prepare_action_maps.py`，按安装中的 ROM/hash/core metadata 可重复生成；`assets/action_catalog.json` 保存逐游戏配置、映射哈希和语义覆盖标记，按系统提供按钮顺序、键盘绑定、单键 action index 和 null 位。使用方式：

```bash
conda activate stable-retro-recording
python -m human_data_recording.prepare_action_maps --check
# 新增 ROM 或人工修改标签后刷新；不覆盖已有标签、不删除旧游戏映射。
python -m human_data_recording.prepare_action_maps --write
```

没有重新安装 Conda 环境。现有 loader 按 game ID 自动读取 map，录制 schema 保持不变。新源码资源可 Git track；ROM 和实际录制产物不提交。上述实现/验证阶段未执行 git add/commit/push；后续提交与推送单独进行。

## 828 款游戏逐一短程录制

工具的 `scripts/audit_ready_games.py` 使用有界、独立子进程，以合成键盘输入运行 NOOP、单按钮和组合动作。4 workers、每游戏最多 60 秒；NES 计划 10 步、Genesis/SNES 计划 14 步，遇到 done 就停止。测试不是新的人类 demonstration。

| 检查 | 实测结果 |
| --- | --- |
| 录制 / 实际按钮布局与 action 编码 | 828 通过 |
| session 静态校验：RGB、live state、动作/语义、reward、计数/哈希 | 828 通过 |
| 默认初始化条件 | 827 默认；NHL94 显式 state override |
| 原生 live state 恢复后原下一步 RGB/RAM | 822 通过、6 不一致 |
| live state custom reset 与独立 NOOP 参考 | 825 通过、3 不一致 |
| 短轨迹 replay / BK2 | 825 通过；另 3 款 replay 在 reset RAM 检查失败，未完成 BK2 检查 |

合计 10375 transitions、11203 PNG、11203 live states（另有每 episode 的 initial.state），约 1.6 GiB、248.96 秒。`DavidCranesAmazingTennis-Genesis` 提前 done；不是所有游戏都完整执行了计划中的每个单键。

NHL94 的 `default_player_state[0]` 多一个句点，指向缺失文件。可显式传 `--state PenguinsVsSenators.start`；本次审计记录了此选择，没有修改 integration，也没有将它当成默认配置通过。

原下一步恢复不一致的 6 款为 `AsterixAndTheGreatRescue-Genesis-v0`、`KidIcarus-Nes-v0`、`SuperDoubleDragon-Snes-v0`、`JikkyouOshaberiParodius-Snes`、`StarFox-Snes`、`SuperMarioWorld2-Snes-v0`。后三款同时出现 custom reset 和 replay 不一致。对 6 款改为单 worker 串行、独立新进程复测，异常均重现；原因尚未进一步定位，不擅自归因为 ROM 损坏或 mapping 错误。原始真实录制的数据保留，不依赖重放补造 state。

全量报告：`/tmp/sretro-action-catalog.Hxiilqyy/all-games/audit.json`；串行复测：同根目录 `retry-serial/audit.json`。这些路径是历史测试产物，不保证持久保存。工具自己的完整回归为 **77 项通过**，wheel 打包检查包含全部 828 个 JSON 和 catalog，不含 ROM/录制数据。

详细操作说明和失败项见工具目录下的 `docs/action_maps.md` 与 `docs/20260907_action_maps_validation.md`。这次覆盖扩展不替代真实桌面长时试玩、整局游戏终止标准、每个按钮的游戏功能查证或不同 core 构建的恢复验证。
