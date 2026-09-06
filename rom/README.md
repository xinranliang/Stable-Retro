# ROM 原始文件库

检查日期：2026-09-05（America/New_York），UTC 时间见 [source_manifest.json](source_manifest.json)。

**后续本地验证和正式导入已完成：** 三个 ZIP 完整性及 SHA-1/MD5/CRC32 全部通过，已正式导入 827 个匹配游戏。加上原有 Airstriker，当前 828 个 ROM 均通过哈希验证，7 个游戏启动抽测通过。见 [20260905 ROM 验证及导入记录](../context/implementation/20260905_rom_validation.md)。下文及 `source_manifest.json` 的访问受限状态保留为最初源站检查记录，不再代表本地文件缺失；原始 ZIP 保持不变。

```text
rom/
  nes/
  snes/
  genesis/
  source_manifest.json
  README.md
```

## 来源检查结果

本次检查的 [Archive.org 文件列表](https://archive.org/download/No-Intro-Collection_2016-01-03_Fixed) 确实列出了三个平台的 ZIP，该来源也被当前 Stable Retro 的 [getting_started.md](../docs/getting_started.md) 引用。但这不能证明其中每个 ROM 都与当前 integration 匹配。

公开元数据把三个文件全部标记为 `private=true`，item 标记为 `access-restricted-item=true`。实际从 canonical 下载 URL 跟随正常重定向后，NES/SNES 的 HEAD 请求返回 401，Genesis 返回 403。当前没有取得任何 ROM 文件，也没有将 HTTP 错误页保存为 ZIP。

| 平台目录 | 来源文件 | 公布大小（bytes） | 下载状态 | Stable Retro stable integration 数 |
| --- | --- | ---: | --- | ---: |
| `nes/` | `Nintendo - Nintendo Entertainment System.zip` | 288252150 | 401，访问受限 | 298 |
| `snes/` | `Nintendo - Super Nintendo Entertainment System.zip` | 3111066120 | 401，访问受限 | 184 |
| `genesis/` | `Sega - Mega Drive - Genesis.zip` | 1165227266 | 403，访问受限 | 338 |

三个包共 4564545536 bytes，约 4.56 GB / 4.25 GiB，仅包含压缩包大小。来源的 filecount 不是本地可运行游戏数，integration 数也不是已验证匹配数。平台内可能有多个地区和 revision，实际覆盖率需要拿到内容后计算。

`source_manifest.json` 保存来源 URL、目标路径、公布的大小/压缩包 SHA-1/MD5/CRC32、访问状态和本地 tag。`null` 校验结果表示尚未检查，不表示校验失败。

## 取得可访问文件后的验证流程

1. 将原始 ZIP 放进对应平台目录，保留原名。若下载同一来源的原包，验证文件大小和公布 SHA-1，再用 `unzip -t` 或 `7z t` 检查压缩内容完整性。使用其他来源或重新压缩的文件时，应另记其来源和 hash，不能拿此处旧 ZIP 的 hash 判断内部 ROM 是否有效。
2. 比较每个 ROM 的标准化 SHA-1 与本地 `stable_retro/data/{stable,experimental,contrib}/<game>/rom.sha`。NES `.nes` 按当前实现跳过 16-byte header；Genesis `.smd` 涉及格式处理，其他普通 cartridge ROM 通常对完整内容计算 SHA-1。优先复用 `groom_rom()` / `verify_hash()` 的校验约定。
3. 在已安装的 recording 环境执行导入。导入器递归读取目录、小写扩展名 `.zip` 和其中的嵌套 ZIP，不要求先解压整个合集。对这些大合集，外层 ZIP 的原始 hash 检查会因 32 MiB 上限被跳过，但脚本仍继续打开 ZIP 并检查成员；成员本身的大小/格式仍需符合导入器要求。其他压缩格式先确认内容再转换或解压。
4. 逐个检查目标 integration 的 `get_romfile_path()` 和 `verify_hash()`。导入器的 Imported 计数不是覆盖率证明，未匹配文件需要单独报告；同一 ROM 可能关联多个 integration，应按目标 game ID 查漏。
5. 对准备录制的游戏分别启动 `make(..., render_mode="rgb_array", record=False)`，执行 reset 和若干 step，记录成功/失败及原因。一个 native core 失败可能影响进程，批量 smoke test 每个游戏使用独立子进程和超时。完成后再手动确认画面/动作/reward。

安装完成后，导入命令为：

```bash
conda activate stable-retro-recording
cd /scratch/gpfs/CHIJ/xinran/projects/game-envalgm/Stable-Retro
python -m stable_retro.import ./rom/nes ./rom/snes ./rom/genesis
```

导入会复制匹配 ROM 到当前加载 package 的 integration 目录，并可能覆盖其中已有 ROM；原始 ZIP 保留。先打印 `stable_retro.__file__` / `stable_retro.data.path()` 确认目标，正式采集日志记录实际 ROM hash、game/state 和 core 版本。

此目录的 `.gitignore` 只保留说明、来源清单和目录占位符，ROM/ZIP 数据默认忽略。导入后的运行副本由 `stable_retro/data/.gitignore` 的 `.nes` / `.sfc` / `.md` 规则忽略；提交代码时仍需检查 `git status`，不要将 ROM 数据强制加入版本控制。

当前继续下载所需的信息：可通过正常权限访问的源文件，或用户已取得并放入上述目录的 ZIP/ROM。登录是否能解除该 item 的限制尚未验证；当前访问状态不能用来判定文件内部 ROM 是有效还是无效。
