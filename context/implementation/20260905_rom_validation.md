# 2026-09-05 ROM 文件验证

本地源码：`v1.0.1`，commit `ec7a62718a1f99f34bf5e5d5c57255c9a53df507`。

最新状态：已完成正式批量导入，新增 827 个匹配游戏；加上原有 Airstriker，当前 828 个 integration 的 ROM 均通过哈希检查。以下先保留导入前的文件验证记录；正式导入结果见文末。

三个 `rom/` ZIP 的实际大小、SHA-1、MD5、CRC32 均与之前保存的 [来源元数据](../../rom/source_manifest.json) 一致。逐个完整读取外层和内层 ZIP 成员，全部 CRC 通过，没有发现加密成员、重复路径、路径穿越或截断。检查期间三个源文件的大小、mtime 和 inode 均未变化；原始 ZIP 未被修改、解压覆盖或正式导入。

| 平台 | 最内层文件数 | CRC 验证成员数（含单游戏 ZIP） | stable integration 哈希覆盖 | experimental 哈希覆盖 |
| --- | ---: | ---: | ---: | ---: |
| NES | 2690 | 5380 | 297 / 298 | 4 / 4 |
| SNES | 3402 | 6804 | 184 / 184 | 4 / 4 |
| Genesis | 1707 | 3414 | 333 / 338 | 5 / 5 |

匹配使用当前源码中原样的 `groom_rom()`：NES 跳过 16-byte header，其他当前成员格式使用完整内容的 SHA-1。比较对象为 integration 的 `rom.sha`，每个 hash 保留所有匹配 integration，不使用导入器会丢失重复目标的单值字典统计覆盖率。结果意味着内容与 integration 预期版本一致，不等于全部游戏均已实际启动。

## 限制与缺口

- NES 包含 2688 个 `.nes` 和 2 个 `.bin`，其中 19 个 `.nes` 的前四字节不是标准 iNES magic `NES\x1a`。这些文件与来源包一致，但不能直接视为标准带头 NES ROM；不要盲目补头，需确认 mapper/格式。本次未修改它们。
- SNES 包含 3392 个 `.sfc` 和 10 个 `.bin`；Genesis 为 1707 个 `.md`。扩展名、合集文件总数均不能替代具体 integration 的哈希检查。
- 很多有效 ROM 没有对应的 Stable Retro integration；未匹配不自动表示损坏。
- Genesis 的 6 个 contrib/custom integrations 均没有匹配该包。

stable 集合缺少以下匹配版本：

```text
SuperMarioBros2Japan-Nes-v0
Airstriker-Genesis-v0
NHL941on1-Genesis-v0
NHL942on2-Genesis-v0
SonicTheHedgehogRandomLevels-Genesis-v0
VirtuaFighter2-Genesis-v0
```

Airstriker 的 ROM 已随仓库提供，因此不依赖本次下载包。其他缺口需要对应 `rom.sha` 的版本，不能仅凭相似文件名强行导入。

## 诊断产物

本次只读扫描脚本和逐文件匹配结果位于临时目录：

- `/tmp/stableretro-rom-validation.GkgVTo/validate.py`
- `/tmp/stableretro-rom-validation.GkgVTo/results.json`

上述临时文件可能被系统清理；本文件保留验证结论。

## 实际启动抽测：通过

新建 `stable-retro-recording` 环境，editable 安装本地 `v1.0.1` 后完成以下测试。每个游戏使用独立子进程，120 步固定输入；下载 ROM 仅复制到各自临时 custom integration，测试结束清理，没有正式导入或覆盖仓库 integration。

| 游戏 | RGB shape | 120 步 reward 总和 |
| --- | --- | ---: |
| Airstriker-Genesis-v0（仓库自带） | 224 × 320 × 3 | 0 |
| SuperMarioBros-Nes-v0 | 224 × 240 × 3 | 172 |
| MegaMan2-Nes-v0 | 224 × 240 × 3 | 155 |
| SuperMarioWorld-Snes-v0 | 224 × 256 × 3 | 0 |
| DonkeyKongCountry-Snes-v0 | 224 × 256 × 3 | 2 |
| SonicTheHedgehog-Genesis-v0 | 224 × 320 × 3 | 0 |
| StreetsOfRage2-Genesis-v0 | 224 × 320 × 3 | 0 |

7/7 游戏均通过 `verify_hash()`、`make/reset/step`、RGB observation space、有限 reward、画面变化和 PNG 编码检查。默认 state 重复 reset 的初始像素一致；运行后用 `em.get_state()` 保存状态并赋给 `initial_state`，重复 reset 的像素也一致（reset 自带一次 no-op）。这不是长时间完整通关测试，也不是所有 814 个匹配 integration 的启动认证。

官方 `RetroInteractive` 在 Xvfb 下运行、绘图 60 步并关闭：通过。实际 `DISPLAY=localhost:10.0` 无法连接，真实手动键盘操作尚未验收。采集工具的逐帧 PNG/action/reward JSONL 功能仍是 working note 中的实现计划，本次没有实现该工具。

启动诊断脚本和详细结果：`/tmp/stableretro-rom-validation.GkgVTo/smoke.py`、`/tmp/stableretro-rom-validation.GkgVTo/smoke_results.json`。

## 正式批量导入：完成

2026-09-05 在 `stable-retro-recording` 环境中执行用户指定命令：

```bash
python -m stable_retro.import ./rom/nes ./rom/snes ./rom/genesis
```

命令正常退出，输出 `Imported 827 games`。导入前仅有自带 Airstriker；本次新增 stable 814 个、experimental 13 个，未覆盖原有游戏 ROM。导入位置是当前 editable package 的 `stable_retro/data/`，没有更改 package 源码或安装依赖。

| 平台 | stable 已安装 / 配置总数 | experimental 已安装 / 配置总数 | 当前已安装合计 |
| --- | ---: | ---: | ---: |
| NES | 297 / 298 | 4 / 4 | 301 |
| SNES | 184 / 184 | 4 / 4 | 188 |
| Genesis | 334 / 338 | 5 / 5 | 339 |
| 合计 | 815 / 820 | 13 / 13 | 828 |

Genesis 的 334 个 stable ROM 包含仓库原有 Airstriker。所有 828 个实际安装 ROM 均通过 `verify_hash()`，无哈希错误。三个源 ZIP 的大小、mtime、inode 与导入前一致，重新计算 SHA-1 也与原来源元数据一致。

再次对上表启动抽测中的 7 个游戏分别运行 120 步，这次直接加载正式安装目录、没有临时复制 integration：全部通过。RGB observation space、有限 reward、画面变化和默认 state 重复 reset 检查正常，测试显式 `record=False`。这仍是抽测，不是全部 828 个游戏的运行认证。

仍未安装的 stable integrations 为 `SuperMarioBros2Japan-Nes-v0`、`NHL941on1-Genesis-v0`、`NHL942on2-Genesis-v0`、`SonicTheHedgehogRandomLevels-Genesis-v0`、`VirtuaFighter2-Genesis-v0`；另有 6 个 Genesis contrib/custom integrations 无匹配版本。未匹配文件保留在原始 ZIP 中，没有强行改名导入，也没有下载额外 ROM。

默认 `make()` 使用 stable 集合。若需要本次导入的 experimental 游戏，例如 `Contra-Nes`，显式使用 `inttype=retro.data.Integrations.EXPERIMENTAL`；已安装不意味着它自动进入默认 stable 列表。

逐游戏路径、哈希验证结果、缺失清单和启动抽测数据保存在 [正式导入结果 JSON](20260905_rom_import_results.json)。本次导入的 `.nes` / `.sfc` / `.md` 已由现有 `stable_retro/data/.gitignore` 排除，未加入 Git。
