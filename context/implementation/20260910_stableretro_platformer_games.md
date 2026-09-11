# Stable-Retro Mario-like 游戏清单

来源：`stable-retro-platform-game-catalog.pdf`（45 页；人工审核记录日期 2026-08-18，图鉴生成日期 2026-08-21）。

仅提取卡片标签为 **YES / MARIO-LIKE** 的条目；保留原图鉴顺序、游戏名称、平台、Stable-Retro ID、候选标记与实验性 integration 标记。未对游戏重新分类。

**2026-09-10 本地 ROM-ready 定稿：原始 141 项中，130 项已就绪，11 项缺少已导入 ROM。** 原始图鉴清单保留如下；本文末尾的“ROM-ready 最终白名单（130 项）”是当前本地可用 ROM 的筛选结果。此处定稿的是 ROM 就绪范围，不是重新认定游戏类型，也不是全部 828 个 ROM-ready 游戏的平台游戏总表。

## 数量与范围

| 平台 | 提取条目数 |
| --- | ---: |
| Game Boy | 10 |
| NES | 131 |
| **合计** | **141** |

原 PDF 共 170 个条目：141 个 YES、28 个 NO、1 个 UNSURE（第 1–2 页）。本清单不包含 NO 和 UNSURE。`Gimmick-Nes-v0` 在第 28 页被标记为 UNSURE / REVIEW，因此未纳入。

计数单位是图鉴条目 / integration，不是去重后的独立作品；不同平台的同名条目保留。141 项中有 129 项 STRICT CANDIDATE、12 项 BOUNDARY CANDIDATE；这两种候选标记不改变本次按 YES 标签筛选的规则。

两项明确标注的 EXPERIMENTAL INTEGRATION 已保留：`BarbieGameGirl-GameBoy`（第 4 页）和 `Contra-Nes`（第 15 页）。这两个 ID 按原文保留，不补加 `-v0`。备注为“未标注”仅表示原卡片没有实验性标记，并非重新核验其状态。

> PDF 第 2 页的定义：YES 表示主要关卡推进依赖角色在平台间跳跃、落地与移动，跳跃时机和落点控制是核心挑战。标签基于名称与截图的人工审核，并非完整通关审查；不代表 RL 可训练性、策略质量或 SFT 价值。

## 文件格式

`stable_retro_mario_like_games.md`：按平台分组的可读清单。

`stable_retro_mario_like_games.json`：包含来源信息、统计与 `games` 数组；每项包含游戏名称、平台、integration ID、原图鉴编号、原始标签、候选标记、实验性标记和页码。

`stable_retro_mario_like_games.txt`：每行一个 integration ID，共 141 行，无标题或注释，表示原始图鉴候选白名单，不等于本地 ROM-ready 白名单。当前本地 ROM-ready 的 130 项以本文末尾列表为准；本次未创建或更新这些配套文件。

## Game Boy（10 项）

| 原图鉴编号 | 游戏名称 | Stable-Retro ID | 候选标记 | 实验性标记 | PDF 页码 |
| ---: | --- | --- | --- | --- | ---: |
| 001 | Adventure Island | `AdventureIsland-GameBoy-v0` | STRICT CANDIDATE | 未标注 | 3 |
| 003 | Alfred Chicken | `AlfredChicken-GameBoy-v0` | STRICT CANDIDATE | 未标注 | 3 |
| 004 | Asterix & Obelix | `AsterixAndObelix-GameBoy-v0` | STRICT CANDIDATE | 未标注 | 3 |
| 005 | Attack of the Killer Tomatoes | `AttackOfTheKillerTomatoes-GameBoy-v0` | STRICT CANDIDATE | 未标注 | 4 |
| 007 | Banishing Racer | `BanishingRacer-GameBoy-v0` | STRICT CANDIDATE | 未标注 | 4 |
| 008 | Barbie - Game Girl | `BarbieGameGirl-GameBoy` | STRICT CANDIDATE | EXPERIMENTAL INTEGRATION | 4 |
| 009 | Bart Simpson's Escape from Camp Deadly | `BartSimpsonsEscapeFromCampDeadly-GameBoy-v0` | STRICT CANDIDATE | 未标注 | 5 |
| 010 | Battletoads in Ragnarok's World | `BattletoadsInRagnaroksWorld-GameBoy-v0` | BOUNDARY CANDIDATE | 未标注 | 5 |
| 013 | The Addams Family | `AddamsFamily-GameBoy-v0` | STRICT CANDIDATE | 未标注 | 6 |
| 014 | The Adventures of Star Saver | `AdventuresOfStarSaver-GameBoy-v0` | STRICT CANDIDATE | 未标注 | 6 |

## NES（131 项）

| 原图鉴编号 | 游戏名称 | Stable-Retro ID | 候选标记 | 实验性标记 | PDF 页码 |
| ---: | --- | --- | --- | --- | ---: |
| 015 | 8 Eyes | `8Eyes-Nes-v0` | STRICT CANDIDATE | 未标注 | 7 |
| 016 | Adventure Island 3 | `AdventureIsland3-Nes-v0` | STRICT CANDIDATE | 未标注 | 7 |
| 017 | Adventure Island II | `AdventureIslandII-Nes-v0` | STRICT CANDIDATE | 未标注 | 7 |
| 018 | Alfred Chicken | `AlfredChicken-Nes-v0` | STRICT CANDIDATE | 未标注 | 7 |
| 019 | Alien 3 | `Alien3-Nes-v0` | STRICT CANDIDATE | 未标注 | 8 |
| 020 | Amagon | `Amagon-Nes-v0` | STRICT CANDIDATE | 未标注 | 8 |
| 022 | Astyanax | `Astyanax-Nes-v0` | STRICT CANDIDATE | 未标注 | 8 |
| 023 | Athena | `Athena-Nes-v0` | STRICT CANDIDATE | 未标注 | 9 |
| 024 | Atlantis no Nazo | `AtlantisNoNazo-Nes-v0` | STRICT CANDIDATE | 未标注 | 9 |
| 025 | Attack of the Killer Tomatoes | `AttackOfTheKillerTomatoes-Nes-v0` | STRICT CANDIDATE | 未标注 | 9 |
| 026 | Balloon Fight | `BalloonFight-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 9 |
| 027 | Banana Prince | `BananaPrince-Nes-v0` | STRICT CANDIDATE | 未标注 | 10 |
| 028 | Barbie | `Barbie-Nes-v0` | STRICT CANDIDATE | 未标注 | 10 |
| 029 | Battletoads | `Battletoads-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 10 |
| 030 | Bio Miracle Bokutte Upa | `BioMiracleBokutteUpa-Nes-v0` | STRICT CANDIDATE | 未标注 | 10 |
| 031 | Bram Stoker's Dracula | `BramStokersDracula-Nes-v0` | STRICT CANDIDATE | 未标注 | 11 |
| 034 | Bucky O'Hare | `BuckyOHare-Nes-v0` | STRICT CANDIDATE | 未标注 | 11 |
| 035 | Captain America and the Avengers | `CaptainAmericaAndTheAvengers-Nes-v0` | STRICT CANDIDATE | 未标注 | 12 |
| 036 | Captain Planet and the Planeteers | `CaptainPlanetAndThePlaneteers-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 12 |
| 037 | Captain Silver | `CaptainSilver-Nes-v0` | STRICT CANDIDATE | 未标注 | 12 |
| 039 | Castlevania | `Castlevania-Nes-v0` | STRICT CANDIDATE | 未标注 | 13 |
| 040 | Castlevania III - Dracula's Curse | `CastlevaniaIIIDraculasCurse-Nes-v0` | STRICT CANDIDATE | 未标注 | 13 |
| 041 | Cat Ninden Teyandee | `CatNindenTeyandee-Nes-v0` | STRICT CANDIDATE | 未标注 | 13 |
| 043 | Challenger | `Challenger-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 14 |
| 044 | Circus Caper | `CircusCaper-Nes-v0` | STRICT CANDIDATE | 未标注 | 14 |
| 047 | Cliffhanger | `Cliffhanger-Nes-v0` | STRICT CANDIDATE | 未标注 | 15 |
| 048 | Conan | `Conan-Nes-v0` | STRICT CANDIDATE | 未标注 | 15 |
| 049 | Conquest of the Crystal Palace | `ConquestOfTheCrystalPalace-Nes-v0` | STRICT CANDIDATE | 未标注 | 15 |
| 050 | Contra | `Contra-Nes` | STRICT CANDIDATE | EXPERIMENTAL INTEGRATION | 15 |
| 051 | Contra Force | `ContraForce-Nes-v0` | STRICT CANDIDATE | 未标注 | 16 |
| 052 | Cross Fire | `CrossFire-Nes-v0` | STRICT CANDIDATE | 未标注 | 16 |
| 053 | Darkman | `Darkman-Nes-v0` | STRICT CANDIDATE | 未标注 | 16 |
| 056 | Dirty Harry | `DirtyHarry-Nes-v0` | STRICT CANDIDATE | 未标注 | 17 |
| 061 | Felix the Cat | `FelixTheCat-Nes-v0` | STRICT CANDIDATE | 未标注 | 18 |
| 062 | Flying Dragon - The Secret Scroll | `FlyingDragonTheSecretScroll-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 18 |
| 063 | Fox's Peter Pan & the Pirates - The Revenge of Captain Hook | `FoxsPeterPanAndThePiratesTheRevengeOfCaptainHook-Nes-v0` | STRICT CANDIDATE | 未标注 | 19 |
| 064 | G.I. Joe - A Real American Hero | `GIJoeARealAmericanHero-Nes-v0` | STRICT CANDIDATE | 未标注 | 19 |
| 065 | G.I. Joe - The Atlantis Factor | `GIJoeTheAtlantisFactor-Nes-v0` | STRICT CANDIDATE | 未标注 | 19 |
| 066 | Ghosts'n Goblins | `GhostsnGoblins-Nes-v0` | STRICT CANDIDATE | 未标注 | 19 |
| 067 | Ghoul School | `GhoulSchool-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 20 |
| 068 | Hammerin' Harry | `HammerinHarry-Nes-v0` | STRICT CANDIDATE | 未标注 | 20 |
| 069 | Hello Kitty World | `HelloKittyWorld-Nes-v0` | STRICT CANDIDATE | 未标注 | 20 |
| 070 | Home Alone 2 - Lost in New York | `HomeAlone2LostInNewYork-Nes-v0` | STRICT CANDIDATE | 未标注 | 20 |
| 072 | IronSword - Wizards & Warriors II | `IronSwordWizardsAndWarriorsII-Nes-v0` | STRICT CANDIDATE | 未标注 | 21 |
| 073 | Jackie Chan's Action Kung Fu | `JackieChansActionKungFu-Nes-v0` | STRICT CANDIDATE | 未标注 | 21 |
| 074 | Jajamaru no Daibouken | `JajamaruNoDaibouken-Nes-v0` | STRICT CANDIDATE | 未标注 | 21 |
| 075 | James Bond Jr | `JamesBondJr-Nes-v0` | STRICT CANDIDATE | 未标注 | 22 |
| 076 | Joe & Mac | `JoeAndMac-Nes-v0` | STRICT CANDIDATE | 未标注 | 22 |
| 077 | Journey to Silius | `JourneyToSilius-Nes-v0` | STRICT CANDIDATE | 未标注 | 22 |
| 079 | Kabuki - Quantum Fighter | `KabukiQuantumFighter-Nes-v0` | STRICT CANDIDATE | 未标注 | 23 |
| 080 | Kaiketsu Yancha Maru 2 - Karakuri Land | `KaiketsuYanchaMaru2KarakuriLand-Nes-v0` | STRICT CANDIDATE | 未标注 | 23 |
| 081 | Kaiketsu Yancha Maru 3 - Taiketsu! Zouringen | `KaiketsuYanchaMaru3TaiketsuZouringen-Nes-v0` | STRICT CANDIDATE | 未标注 | 23 |
| 082 | Kamen no Ninja - Akakage | `KamenNoNinjaAkakage-Nes-v0` | STRICT CANDIDATE | 未标注 | 23 |
| 083 | Kanshakudama Nage Kantarou no Toukaidou Gojuusan Tsugi | `KanshakudamaNageKantarouNoToukaidouGojuusanTsugi-Nes-v0` | STRICT CANDIDATE | 未标注 | 24 |
| 084 | Kid Icarus | `KidIcarus-Nes-v0` | STRICT CANDIDATE | 未标注 | 24 |
| 085 | Kid Klown in Night Mayor World | `KidKlownInNightMayorWorld-Nes-v0` | STRICT CANDIDATE | 未标注 | 24 |
| 086 | Kid Niki - Radical Ninja | `KidNikiRadicalNinja-Nes-v0` | STRICT CANDIDATE | 未标注 | 24 |
| 087 | Kirby's Adventure | `KirbysAdventure-Nes-v0` | STRICT CANDIDATE | 未标注 | 25 |
| 088 | Low G Man - The Low Gravity Man | `LowGManTheLowGravityMan-Nes-v0` | STRICT CANDIDATE | 未标注 | 25 |
| 089 | M.C. Kids | `MCKids-Nes-v0` | STRICT CANDIDATE | 未标注 | 25 |
| 091 | Mario Bros. | `MarioBros-Nes-v0` | STRICT CANDIDATE | 未标注 | 26 |
| 092 | Mega Man | `MegaMan-Nes-v0` | STRICT CANDIDATE | 未标注 | 26 |
| 093 | Mega Man 2 | `MegaMan2-Nes-v0` | STRICT CANDIDATE | 未标注 | 26 |
| 094 | Metal Storm | `MetalStorm-Nes-v0` | STRICT CANDIDATE | 未标注 | 26 |
| 095 | Mickey Mousecapade | `MickeyMousecapade-Nes-v0` | STRICT CANDIDATE | 未标注 | 27 |
| 096 | Mighty Bomb Jack | `MightyBombJack-Nes-v0` | STRICT CANDIDATE | 未标注 | 27 |
| 097 | Mitsume ga Tooru | `MitsumeGaTooru-Nes-v0` | STRICT CANDIDATE | 未标注 | 27 |
| 098 | Monster in My Pocket | `MonsterInMyPocket-Nes-v0` | STRICT CANDIDATE | 未标注 | 27 |
| 099 | Monster Party | `MonsterParty-Nes-v0` | STRICT CANDIDATE | 未标注 | 28 |
| 101 | Mystery Quest | `MysteryQuest-Nes-v0` | STRICT CANDIDATE | 未标注 | 28 |
| 102 | Ninja Crusaders | `NinjaCrusaders-Nes-v0` | STRICT CANDIDATE | 未标注 | 28 |
| 103 | Ninja Gaiden | `NinjaGaiden-Nes-v0` | STRICT CANDIDATE | 未标注 | 29 |
| 104 | Ninja Gaiden II - The Dark Sword of Chaos | `NinjaGaidenIITheDarkSwordOfChaos-Nes-v0` | STRICT CANDIDATE | 未标注 | 29 |
| 105 | Ninja Gaiden III - The Ancient Ship of Doom | `NinjaGaidenIIITheAncientShipOfDoom-Nes-v0` | STRICT CANDIDATE | 未标注 | 29 |
| 107 | Noah's Ark | `NoahsArk-Nes-v0` | STRICT CANDIDATE | 未标注 | 30 |
| 108 | Panic Restaurant | `PanicRestaurant-Nes-v0` | STRICT CANDIDATE | 未标注 | 30 |
| 109 | Pizza Pop! | `PizzaPop-Nes-v0` | STRICT CANDIDATE | 未标注 | 30 |
| 111 | Puss 'n Boots - Pero's Great Adventure | `PussNBootsPerosGreatAdventure-Nes-v0` | STRICT CANDIDATE | 未标注 | 31 |
| 114 | RoboCop 2 | `RoboCop2-Nes-v0` | STRICT CANDIDATE | 未标注 | 31 |
| 115 | RoboCop 3 | `RoboCop3-Nes-v0` | STRICT CANDIDATE | 未标注 | 32 |
| 116 | Rockin' Kats | `RockinKats-Nes-v0` | STRICT CANDIDATE | 未标注 | 32 |
| 117 | Rollergames | `Rollergames-Nes-v0` | STRICT CANDIDATE | 未标注 | 32 |
| 118 | Rush'n Attack | `RushnAttack-Nes-v0` | STRICT CANDIDATE | 未标注 | 32 |
| 119 | SD Hero Soukessen - Taose! Aku no Gundan | `SDHeroSoukessenTaoseAkuNoGundan-Nes-v0` | STRICT CANDIDATE | 未标注 | 33 |
| 120 | Seikima II - Akuma no Gyakushuu! | `SeikimaIIAkumaNoGyakushuu-Nes-v0` | STRICT CANDIDATE | 未标注 | 33 |
| 121 | Shatterhand | `Shatterhand-Nes-v0` | STRICT CANDIDATE | 未标注 | 33 |
| 124 | Son Son | `SonSon-Nes-v0` | STRICT CANDIDATE | 未标注 | 34 |
| 125 | Spelunker | `Spelunker-Nes-v0` | STRICT CANDIDATE | 未标注 | 34 |
| 126 | Star Wars | `StarWars-Nes-v0` | STRICT CANDIDATE | 未标注 | 34 |
| 128 | Super C | `SuperC-Nes-v0` | STRICT CANDIDATE | 未标注 | 35 |
| 129 | Super Mario Bros. | `SuperMarioBros-Nes-v0` | STRICT CANDIDATE | 未标注 | 35 |
| 130 | Super Mario Bros. 2 (Japan) | `SuperMarioBros2Japan-Nes-v0` | STRICT CANDIDATE | 未标注 | 35 |
| 131 | Super Mario Bros. 3 | `SuperMarioBros3-Nes-v0` | STRICT CANDIDATE | 未标注 | 36 |
| 132 | Super Pitfall | `SuperPitfall-Nes-v0` | STRICT CANDIDATE | 未标注 | 36 |
| 133 | Swamp Thing | `SwampThing-Nes-v0` | STRICT CANDIDATE | 未标注 | 36 |
| 134 | Takahashi Meijin no Bugutte Honey | `TakahashiMeijinNoBugutteHoney-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 36 |
| 135 | Teenage Mutant Ninja Turtles | `TeenageMutantNinjaTurtles-Nes-v0` | STRICT CANDIDATE | 未标注 | 37 |
| 136 | Terminator 2 - Judgment Day | `Terminator2JudgmentDay-Nes-v0` | STRICT CANDIDATE | 未标注 | 37 |
| 137 | Tetsuwan Atom | `TetsuwanAtom-Nes-v0` | STRICT CANDIDATE | 未标注 | 37 |
| 138 | The Addams Family | `AddamsFamily-Nes-v0` | STRICT CANDIDATE | 未标注 | 37 |
| 139 | The Addams Family - Pugsley's Scavenger Hunt | `AddamsFamilyPugsleysScavengerHunt-Nes-v0` | STRICT CANDIDATE | 未标注 | 38 |
| 140 | The Adventures of Rocky and Bullwinkle and Friends | `AdventuresOfRockyAndBullwinkleAndFriends-Nes-v0` | STRICT CANDIDATE | 未标注 | 38 |
| 141 | The Bugs Bunny Birthday Blowout | `BugsBunnyBirthdayBlowout-Nes-v0` | STRICT CANDIDATE | 未标注 | 38 |
| 142 | The Flintstones - The Rescue of Dino & Hoppy | `FlintstonesTheRescueOfDinoAndHoppy-Nes-v0` | STRICT CANDIDATE | 未标注 | 38 |
| 143 | The Jetsons - Cogswell's Caper | `JetsonsCogswellsCaper-Nes-v0` | STRICT CANDIDATE | 未标注 | 39 |
| 144 | The Jungle Book | `JungleBook-Nes-v0` | STRICT CANDIDATE | 未标注 | 39 |
| 145 | The Legend of Kage | `LegendOfKage-Nes-v0` | STRICT CANDIDATE | 未标注 | 39 |
| 146 | The Legend of Prince Valiant | `LegendOfPrinceValiant-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 39 |
| 148 | The Simpsons - Bart vs. the World | `SimpsonsBartVsTheWorld-Nes-v0` | STRICT CANDIDATE | 未标注 | 40 |
| 149 | The Simpsons - Bartman Meets Radioactive Man | `SimpsonsBartmanMeetsRadioactiveMan-Nes-v0` | STRICT CANDIDATE | 未标注 | 40 |
| 150 | The Smurfs | `Smurfs-Nes-v0` | STRICT CANDIDATE | 未标注 | 40 |
| 151 | The Trolls in Crazyland | `TrollsInCrazyland-Nes-v0` | STRICT CANDIDATE | 未标注 | 41 |
| 152 | The Young Indiana Jones Chronicles | `YoungIndianaJonesChronicles-Nes-v0` | STRICT CANDIDATE | 未标注 | 41 |
| 153 | Thexder | `Thexder-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 41 |
| 154 | Time Zone | `TimeZone-Nes-v0` | STRICT CANDIDATE | 未标注 | 41 |
| 155 | Tiny Toon Adventures | `TinyToonAdventures-Nes-v0` | STRICT CANDIDATE | 未标注 | 42 |
| 156 | Toki | `Toki-Nes-v0` | STRICT CANDIDATE | 未标注 | 42 |
| 157 | Total Recall | `TotalRecall-Nes-v0` | STRICT CANDIDATE | 未标注 | 42 |
| 158 | Totally Rad | `TotallyRad-Nes-v0` | STRICT CANDIDATE | 未标注 | 42 |
| 159 | Treasure Master | `TreasureMaster-Nes-v0` | STRICT CANDIDATE | 未标注 | 43 |
| 160 | Trojan | `Trojan-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 43 |
| 161 | Urusei Yatsura - Lum no Wedding Bell | `UruseiYatsuraLumNoWeddingBell-Nes-v0` | STRICT CANDIDATE | 未标注 | 43 |
| 162 | Vice - Project Doom | `ViceProjectDoom-Nes-v0` | BOUNDARY CANDIDATE | 未标注 | 43 |
| 163 | Wayne's World | `WaynesWorld-Nes-v0` | STRICT CANDIDATE | 未标注 | 44 |
| 164 | Widget | `Widget-Nes-v0` | STRICT CANDIDATE | 未标注 | 44 |
| 165 | Wizards & Warriors | `WizardsAndWarriors-Nes-v0` | STRICT CANDIDATE | 未标注 | 44 |
| 166 | Wrath of the Black Manta | `WrathOfTheBlackManta-Nes-v0` | STRICT CANDIDATE | 未标注 | 44 |
| 167 | Wrecking Crew | `WreckingCrew-Nes-v0` | STRICT CANDIDATE | 未标注 | 45 |
| 168 | Xexyz | `Xexyz-Nes-v0` | STRICT CANDIDATE | 未标注 | 45 |
| 169 | Youkai Club | `YoukaiClub-Nes-v0` | STRICT CANDIDATE | 未标注 | 45 |
| 170 | Youkai Douchuuki | `YoukaiDouchuuki-Nes-v0` | STRICT CANDIDATE | 未标注 | 45 |

## ROM-ready 核验与定稿（2026-09-10）

核验对象是上方原始表格的 141 个唯一 integration ID，而不是根据名称或先前缓存推断。使用 `stable-retro-recording` conda 环境，加载本仓库的 `stable_retro`，通过采集工具的只读 `human_data_recording.prepare_action_maps.scan_roms()` 实际扫描本地已导入 ROM，并与原始清单取交集。

ROM-ready 标准：integration 存在；Stable Retro 能找到已导入的 ROM；按导入器 `groom_rom()` 的规范化规则计算 SHA-1，并与对应 integration 的 `rom.sha` 匹配（NES 不将前 16 字节 iNES header 纳入哈希）。仅在 `rom/` 内有 ZIP，不算已经导入或 ROM-ready。

| 平台 | 原始清单条目 | ROM-ready | 缺少已导入 ROM | 哈希不匹配 | 未识别 ID |
| --- | ---: | ---: | ---: | ---: | ---: |
| Game Boy | 10 | 0 | 10 | 0 | 0 |
| NES | 131 | 130 | 1 | 0 | 0 |
| **合计** | **141** | **130** | **11** | **0** | **0** |

130 项全部是 NES integration，其中 129 项为 `stable`，1 项为 `experimental`：`Contra-Nes`。该 ID 保留原样，不添加 `-v0`；使用 Stable Retro API 时需包含 experimental integrations，例如 `inttype=stable_retro.data.Integrations.ALL`。

这 130 项的模拟器 core、默认 state、`data.json` 和 `scenario.json` 均存在，静态依赖检查全部通过。本次未逐个启动 emulator、打开手动游玩窗口或执行 recording/replay，因此该结论不代表 130 项均通过了本次运行时或录制测试，也不保证每个游戏的 reward / game-over 判定符合采集目标。

同期全仓库扫描仍有 828 个 ROM-ready 游戏 ID（NES 301、SNES 188、Genesis 339），未发现已导入 ROM / 配置校验失败项。本节的 130 项只是上述 141 项 Mario-like 图鉴清单与本地 ROM-ready 集合的交集。

### 未纳入最终白名单的 11 项

以下 ID 的 integration 均存在，但缺少已导入 ROM；本次没有下载或导入 ROM。后续补齐符合各自 `rom.sha` 的 ROM 并导入后，需要重新核验再加入白名单。

```text
AdventureIsland-GameBoy-v0
AlfredChicken-GameBoy-v0
AsterixAndObelix-GameBoy-v0
AttackOfTheKillerTomatoes-GameBoy-v0
BanishingRacer-GameBoy-v0
BarbieGameGirl-GameBoy
BartSimpsonsEscapeFromCampDeadly-GameBoy-v0
BattletoadsInRagnaroksWorld-GameBoy-v0
AddamsFamily-GameBoy-v0
AdventuresOfStarSaver-GameBoy-v0
SuperMarioBros2Japan-Nes-v0
```

### ROM-ready 最终白名单（130 项）

以下按原图鉴顺序列出全部 130 个 ID；一行一个 ID，无重复、无注释，可复制为当前 ROM-ready 白名单。游戏名称、原图鉴编号、页码和 STRICT / BOUNDARY 标记仍可在上方原始表格中按 ID 查询；此次未改变这些分类。

```text
8Eyes-Nes-v0
AdventureIsland3-Nes-v0
AdventureIslandII-Nes-v0
AlfredChicken-Nes-v0
Alien3-Nes-v0
Amagon-Nes-v0
Astyanax-Nes-v0
Athena-Nes-v0
AtlantisNoNazo-Nes-v0
AttackOfTheKillerTomatoes-Nes-v0
BalloonFight-Nes-v0
BananaPrince-Nes-v0
Barbie-Nes-v0
Battletoads-Nes-v0
BioMiracleBokutteUpa-Nes-v0
BramStokersDracula-Nes-v0
BuckyOHare-Nes-v0
CaptainAmericaAndTheAvengers-Nes-v0
CaptainPlanetAndThePlaneteers-Nes-v0
CaptainSilver-Nes-v0
Castlevania-Nes-v0
CastlevaniaIIIDraculasCurse-Nes-v0
CatNindenTeyandee-Nes-v0
Challenger-Nes-v0
CircusCaper-Nes-v0
Cliffhanger-Nes-v0
Conan-Nes-v0
ConquestOfTheCrystalPalace-Nes-v0
Contra-Nes
ContraForce-Nes-v0
CrossFire-Nes-v0
Darkman-Nes-v0
DirtyHarry-Nes-v0
FelixTheCat-Nes-v0
FlyingDragonTheSecretScroll-Nes-v0
FoxsPeterPanAndThePiratesTheRevengeOfCaptainHook-Nes-v0
GIJoeARealAmericanHero-Nes-v0
GIJoeTheAtlantisFactor-Nes-v0
GhostsnGoblins-Nes-v0
GhoulSchool-Nes-v0
HammerinHarry-Nes-v0
HelloKittyWorld-Nes-v0
HomeAlone2LostInNewYork-Nes-v0
IronSwordWizardsAndWarriorsII-Nes-v0
JackieChansActionKungFu-Nes-v0
JajamaruNoDaibouken-Nes-v0
JamesBondJr-Nes-v0
JoeAndMac-Nes-v0
JourneyToSilius-Nes-v0
KabukiQuantumFighter-Nes-v0
KaiketsuYanchaMaru2KarakuriLand-Nes-v0
KaiketsuYanchaMaru3TaiketsuZouringen-Nes-v0
KamenNoNinjaAkakage-Nes-v0
KanshakudamaNageKantarouNoToukaidouGojuusanTsugi-Nes-v0
KidIcarus-Nes-v0
KidKlownInNightMayorWorld-Nes-v0
KidNikiRadicalNinja-Nes-v0
KirbysAdventure-Nes-v0
LowGManTheLowGravityMan-Nes-v0
MCKids-Nes-v0
MarioBros-Nes-v0
MegaMan-Nes-v0
MegaMan2-Nes-v0
MetalStorm-Nes-v0
MickeyMousecapade-Nes-v0
MightyBombJack-Nes-v0
MitsumeGaTooru-Nes-v0
MonsterInMyPocket-Nes-v0
MonsterParty-Nes-v0
MysteryQuest-Nes-v0
NinjaCrusaders-Nes-v0
NinjaGaiden-Nes-v0
NinjaGaidenIITheDarkSwordOfChaos-Nes-v0
NinjaGaidenIIITheAncientShipOfDoom-Nes-v0
NoahsArk-Nes-v0
PanicRestaurant-Nes-v0
PizzaPop-Nes-v0
PussNBootsPerosGreatAdventure-Nes-v0
RoboCop2-Nes-v0
RoboCop3-Nes-v0
RockinKats-Nes-v0
Rollergames-Nes-v0
RushnAttack-Nes-v0
SDHeroSoukessenTaoseAkuNoGundan-Nes-v0
SeikimaIIAkumaNoGyakushuu-Nes-v0
Shatterhand-Nes-v0
SonSon-Nes-v0
Spelunker-Nes-v0
StarWars-Nes-v0
SuperC-Nes-v0
SuperMarioBros-Nes-v0
SuperMarioBros3-Nes-v0
SuperPitfall-Nes-v0
SwampThing-Nes-v0
TakahashiMeijinNoBugutteHoney-Nes-v0
TeenageMutantNinjaTurtles-Nes-v0
Terminator2JudgmentDay-Nes-v0
TetsuwanAtom-Nes-v0
AddamsFamily-Nes-v0
AddamsFamilyPugsleysScavengerHunt-Nes-v0
AdventuresOfRockyAndBullwinkleAndFriends-Nes-v0
BugsBunnyBirthdayBlowout-Nes-v0
FlintstonesTheRescueOfDinoAndHoppy-Nes-v0
JetsonsCogswellsCaper-Nes-v0
JungleBook-Nes-v0
LegendOfKage-Nes-v0
LegendOfPrinceValiant-Nes-v0
SimpsonsBartVsTheWorld-Nes-v0
SimpsonsBartmanMeetsRadioactiveMan-Nes-v0
Smurfs-Nes-v0
TrollsInCrazyland-Nes-v0
YoungIndianaJonesChronicles-Nes-v0
Thexder-Nes-v0
TimeZone-Nes-v0
TinyToonAdventures-Nes-v0
Toki-Nes-v0
TotalRecall-Nes-v0
TotallyRad-Nes-v0
TreasureMaster-Nes-v0
Trojan-Nes-v0
UruseiYatsuraLumNoWeddingBell-Nes-v0
ViceProjectDoom-Nes-v0
WaynesWorld-Nes-v0
Widget-Nes-v0
WizardsAndWarriors-Nes-v0
WrathOfTheBlackManta-Nes-v0
WreckingCrew-Nes-v0
Xexyz-Nes-v0
YoukaiClub-Nes-v0
YoukaiDouchuuki-Nes-v0
```

## Train / ID / OOD split proposal（2026-09-11）

本节记录针对上方 130 个 ROM-ready 游戏 ID 的划分建议：**Train 100 / ID-like 20 / OOD 10**。保持上一轮 proposal 的成员不变，不重新筛选或修改图鉴分类。本次只将方案写入 working note，未生成独立 split 配置文件，未修改采集、训练或评测代码。

### 划分口径与适用范围

- **Train：100 个游戏。** 用于训练数据采集；保留 OOD 对应的训练侧作品以及较广的其他作品覆盖。
- **ID-like：20 个同类未见游戏。** 表示 `same-category held-out` 测试集，不是模型已经见过这些游戏的 seen-game ID 测试。真正的 seen-game ID 评测需要从 Train 游戏中另外留出完整 episodes，不改变这里的 100 / 20 / 10 游戏级配额。
- **OOD：10 个配对迁移测试游戏。** 主要测量同系列未见作品的迁移能力，命名为 `paired-transfer test` 更准确；不是未见系列泛化，也不能仅凭这个标签认定它比分布内测试更难。
- 固定 10 个 OOD 名额时，优先选择 **9 组真实系列留出 + 1 个用户指定的 Captain 配对**。清单中的多作品系列超过 10 组，因此本方案不是“所有系列都必须拆到 OOD”的全局硬约束。Addams Family、Simpsons、Contra 等系列在本方案中保留于 Train，不额外拆出 OOD。
- `CaptainAmericaAndTheAvengers-Nes-v0` 与 `CaptainPlanetAndThePlaneteers-Nes-v0` 是不同作品，不是同一游戏的不同版本。本方案保留用户希望将两者分别放入 Train / OOD 的安排，但显式标记为指定配对例外，不将名称中的 `Captain` 当作系列关系依据。参考：[Captain America 资料](https://www.gamesdatabase.org/game/nintendo-nes/captain-america-and-the-avengers.aspx)、[Captain Planet 资料](https://gamefaqs.gamespot.com/nes/587172-captain-planet-and-the-planeteers/data)。

本方案是实验设计 proposal，不是根据模型分数挑选的测试集，也没有逐个重新测量游戏难度、录制难度或分布距离。建议对外描述为 **100 train / 20 same-category held-out / 10 paired-transfer test**，避免将 ID / OOD 的简写误解为已验证的统计分布性质。

### 数量与原图鉴标签分布

尽量保持原始 STRICT / BOUNDARY 候选标签比例，避免将边界类型集中到某个 split；这不是新的游戏分类。

| Split | 游戏数 | STRICT CANDIDATE | BOUNDARY CANDIDATE | stable | experimental |
| --- | ---: | ---: | ---: | ---: | ---: |
| Train | 100 | 92 | 8 | 99 | 1 |
| ID-like | 20 | 18 | 2 | 20 | 0 |
| OOD | 10 | 9 | 1 | 10 | 0 |
| **合计** | **130** | **119** | **11** | **129** | **1** |

唯一 experimental integration `Contra-Nes` 留在 Train。两个测试集均为 stable integrations；这只描述 integration 来源，不意味着其 reward / game-over 判定或实际录制流程已经全部通过运行时测试。

### OOD 配对依据

优先选择有训练侧参照作品的系列，便于分析训练过其他作品之后的迁移效果。表中的 Train 列是参照作品，不是对应系列在 Train 中的穷尽清单；完整成员以各 split 的代码块为准。

| 留在 Train 的参照作品 | 放入 OOD 的游戏 | 分组依据 |
| --- | --- | --- |
| `SuperMarioBros-Nes-v0` | `SuperMarioBros3-Nes-v0` | Super Mario 系列 |
| `MegaMan-Nes-v0` | `MegaMan2-Nes-v0` | Mega Man 系列 |
| `Castlevania-Nes-v0` | `CastlevaniaIIIDraculasCurse-Nes-v0` | Castlevania 系列 |
| `NinjaGaiden-Nes-v0`、`NinjaGaidenIITheDarkSwordOfChaos-Nes-v0` | `NinjaGaidenIIITheAncientShipOfDoom-Nes-v0` | Ninja Gaiden 系列 |
| `AdventureIslandII-Nes-v0` | `AdventureIsland3-Nes-v0` | Adventure Island 系列 |
| `KaiketsuYanchaMaru2KarakuriLand-Nes-v0` | `KaiketsuYanchaMaru3TaiketsuZouringen-Nes-v0` | Kaiketsu Yancha Maru 系列 |
| `GIJoeARealAmericanHero-Nes-v0` | `GIJoeTheAtlantisFactor-Nes-v0` | G.I. Joe 系列 |
| `RoboCop2-Nes-v0` | `RoboCop3-Nes-v0` | RoboCop 系列 |
| `WizardsAndWarriors-Nes-v0` | `IronSwordWizardsAndWarriorsII-Nes-v0` | Wizards & Warriors 系列 |
| `CaptainAmericaAndTheAvengers-Nes-v0` | `CaptainPlanetAndThePlaneteers-Nes-v0` | 用户指定的跨作品配对，不是同系列 |

### OOD 完整名单（10 个）

```text
SuperMarioBros3-Nes-v0
MegaMan2-Nes-v0
CastlevaniaIIIDraculasCurse-Nes-v0
NinjaGaidenIIITheAncientShipOfDoom-Nes-v0
AdventureIsland3-Nes-v0
KaiketsuYanchaMaru3TaiketsuZouringen-Nes-v0
GIJoeTheAtlantisFactor-Nes-v0
RoboCop3-Nes-v0
IronSwordWizardsAndWarriorsII-Nes-v0
CaptainPlanetAndThePlaneteers-Nes-v0
```

### ID-like 完整名单（20 个）

作为同类未见游戏测试集，保留不同作品和角色的覆盖。其中 `Challenger-Nes-v0`、`ViceProjectDoom-Nes-v0` 是原图鉴中的 BOUNDARY 条目，其余 18 个为 STRICT。它们的 ID-like 身份是本方案的设计假设，不是经过实际轨迹分布检验后的结论。

```text
AlfredChicken-Nes-v0
BananaPrince-Nes-v0
BuckyOHare-Nes-v0
ConquestOfTheCrystalPalace-Nes-v0
HammerinHarry-Nes-v0
JackieChansActionKungFu-Nes-v0
JoeAndMac-Nes-v0
KabukiQuantumFighter-Nes-v0
MCKids-Nes-v0
MitsumeGaTooru-Nes-v0
MonsterInMyPocket-Nes-v0
PanicRestaurant-Nes-v0
RockinKats-Nes-v0
Shatterhand-Nes-v0
TinyToonAdventures-Nes-v0
JungleBook-Nes-v0
Smurfs-Nes-v0
Toki-Nes-v0
Challenger-Nes-v0
ViceProjectDoom-Nes-v0
```

### Train 完整名单（100 个）

```text
8Eyes-Nes-v0
AdventureIslandII-Nes-v0
Alien3-Nes-v0
Amagon-Nes-v0
Astyanax-Nes-v0
Athena-Nes-v0
AtlantisNoNazo-Nes-v0
AttackOfTheKillerTomatoes-Nes-v0
BalloonFight-Nes-v0
Barbie-Nes-v0
Battletoads-Nes-v0
BioMiracleBokutteUpa-Nes-v0
BramStokersDracula-Nes-v0
CaptainAmericaAndTheAvengers-Nes-v0
CaptainSilver-Nes-v0
Castlevania-Nes-v0
CatNindenTeyandee-Nes-v0
CircusCaper-Nes-v0
Cliffhanger-Nes-v0
Conan-Nes-v0
Contra-Nes
ContraForce-Nes-v0
CrossFire-Nes-v0
Darkman-Nes-v0
DirtyHarry-Nes-v0
FelixTheCat-Nes-v0
FlyingDragonTheSecretScroll-Nes-v0
FoxsPeterPanAndThePiratesTheRevengeOfCaptainHook-Nes-v0
GIJoeARealAmericanHero-Nes-v0
GhostsnGoblins-Nes-v0
GhoulSchool-Nes-v0
HelloKittyWorld-Nes-v0
HomeAlone2LostInNewYork-Nes-v0
JajamaruNoDaibouken-Nes-v0
JamesBondJr-Nes-v0
JourneyToSilius-Nes-v0
KaiketsuYanchaMaru2KarakuriLand-Nes-v0
KamenNoNinjaAkakage-Nes-v0
KanshakudamaNageKantarouNoToukaidouGojuusanTsugi-Nes-v0
KidIcarus-Nes-v0
KidKlownInNightMayorWorld-Nes-v0
KidNikiRadicalNinja-Nes-v0
KirbysAdventure-Nes-v0
LowGManTheLowGravityMan-Nes-v0
MarioBros-Nes-v0
MegaMan-Nes-v0
MetalStorm-Nes-v0
MickeyMousecapade-Nes-v0
MightyBombJack-Nes-v0
MonsterParty-Nes-v0
MysteryQuest-Nes-v0
NinjaCrusaders-Nes-v0
NinjaGaiden-Nes-v0
NinjaGaidenIITheDarkSwordOfChaos-Nes-v0
NoahsArk-Nes-v0
PizzaPop-Nes-v0
PussNBootsPerosGreatAdventure-Nes-v0
RoboCop2-Nes-v0
Rollergames-Nes-v0
RushnAttack-Nes-v0
SDHeroSoukessenTaoseAkuNoGundan-Nes-v0
SeikimaIIAkumaNoGyakushuu-Nes-v0
SonSon-Nes-v0
Spelunker-Nes-v0
StarWars-Nes-v0
SuperC-Nes-v0
SuperMarioBros-Nes-v0
SuperPitfall-Nes-v0
SwampThing-Nes-v0
TakahashiMeijinNoBugutteHoney-Nes-v0
TeenageMutantNinjaTurtles-Nes-v0
Terminator2JudgmentDay-Nes-v0
TetsuwanAtom-Nes-v0
AddamsFamily-Nes-v0
AddamsFamilyPugsleysScavengerHunt-Nes-v0
AdventuresOfRockyAndBullwinkleAndFriends-Nes-v0
BugsBunnyBirthdayBlowout-Nes-v0
FlintstonesTheRescueOfDinoAndHoppy-Nes-v0
JetsonsCogswellsCaper-Nes-v0
LegendOfKage-Nes-v0
LegendOfPrinceValiant-Nes-v0
SimpsonsBartVsTheWorld-Nes-v0
SimpsonsBartmanMeetsRadioactiveMan-Nes-v0
TrollsInCrazyland-Nes-v0
YoungIndianaJonesChronicles-Nes-v0
Thexder-Nes-v0
TimeZone-Nes-v0
TotalRecall-Nes-v0
TotallyRad-Nes-v0
TreasureMaster-Nes-v0
Trojan-Nes-v0
UruseiYatsuraLumNoWeddingBell-Nes-v0
WaynesWorld-Nes-v0
Widget-Nes-v0
WizardsAndWarriors-Nes-v0
WrathOfTheBlackManta-Nes-v0
WreckingCrew-Nes-v0
Xexyz-Nes-v0
YoukaiClub-Nes-v0
YoukaiDouchuuki-Nes-v0
```

### 采集、评测与防泄漏约束

1. **按 game ID 整体归属 split。** 同一测试游戏的全部关卡、preloaded / custom states、录制轨迹和 replay 派生样本均归属于该测试集，不得进入训练集。
2. **测试数据不参与训练或调参。** 如果使用 20 个 ID-like 游戏选择 checkpoint、提示词或超参数，应将其改称 validation，并另行准备未参与选择的最终测试集，不能继续作为无偏的最终 test 报告。
3. **另外准备 seen-game ID 评测。** 从 Train 游戏中留出完整 episodes / recording sessions，而不是随机拆分同一轨迹的相邻帧。录制与 replay 产生的同源数据必须一起划分；还应检查不同 sessions 是否包含重复轨迹。该轨迹级划分不改变这里的 100 / 20 / 10 游戏级数量。
4. **固定并记录评测初始化。** 记录实际 game ID、ROM 哈希、initial state 标识 / 哈希及关卡信息，避免后续改变初始化范围而仍按同一任务比较。
5. **分别报告测试结果。** ID-like 与 paired-transfer OOD 分开报告；Captain 指定配对与 9 个同系列留出条目的性质不同，应保留标记。不能仅凭 split 名称假设 OOD 必然更难。

### 名单一致性验证

文档更新后需从各 split 的完整名单代码块解析 ID，并进行以下检查；此验证不替代 emulator / recording / replay 的运行时测试：

- 数量严格为 Train 100、ID-like 20、OOD 10，split 内无重复，split 之间无交集。
- 三个 split 的并集恰好等于上方 130 项 ROM-ready 白名单，不包含 11 项缺少 ROM 的候选。
- 原图鉴 STRICT / BOUNDARY 标签统计及 integration 类型统计与本节汇总表一致。
- OOD 配对表中所有训练侧参照作品属于 Train，所有测试侧作品属于 OOD；配对表恰好覆盖 10 个 OOD ID。
- 原始 141 项图鉴清单及 130 项 ROM-ready 白名单保持不变。

2026-09-11 验证结果：以上检查全部通过；三个 split 的成员和顺序与上一轮 proposal 一致，原有文档内容的 SHA-256 校验确认未变。另用 `stable-retro-recording` 环境的 `stable_retro.data.verify_hash(..., Integrations.ALL)` 重新检查了这 130 个已导入 ROM，全部通过；按当前 ROM SHA-1 检查，未发现跨 split 的完全相同 ROM。该检查不排除不同 ROM 间的内容相似性，也不替代轨迹去重或运行时验证。
