# [MK1 / 01006560184E6000] Mac (Apple M1, 0.2.6)：角色渲染为黑色剪影 / 消失，持续 1 秒以上卡帧

> 状态：草稿，未提交。所有数字均取自本机日志与系统输出，未经推断。

## 环境

| 项目 | 值 |
|---|---|
| S2Arm | 0.2.6 (build 1121)，官方 Developer ID 签名 |
| 机器 | iMac21,1，Apple M1，16 GB 统一内存 |
| 系统 | macOS 26.5.1 (25F80) |
| 显示 | Retina 5120 × 2880（逻辑 2560 × 1440） |
| 游戏 | Mortal Kombat™ 1，v1.27.0，程序号 `01006560184E6000` |
| 安装内容 | 本体 6.3 GB + v1.27 更新 43.5 GB + 17 个 DLC；`build mac` 产物 43.7 GB |
| 启动设置 | 全缺省：`docked=docked`、`fps60=true`、`enhance=false`、`widescreen=true`、`widenForce=false` |

## 复现步骤

1. `s2armctl games "<含本体/更新/DLC 的文件夹>"` → 识别为一条 `Mortal Kombat™ 1（版本 1.27.0）`
2. `s2armctl build 01006560184E6000 mac` → 759 秒完成，`hostFailed: []`
3. `s2armctl launch 01006560184E6000`
4. 首轮在 `frame 4653` 触发 `GPU 挂死安全停（kept the GPU busy 23255 ms, limit 1000 ms）` 后自动重启；第二轮可进入角色选择并进入对战

## 实际现象（玩家反馈「画面根本没法正常玩」）

- 对战画面里角色呈**纯黑剪影或直接不显示**，可见场景、血条、UI，但看不到正常渲染的人物模型
- 画面随后转为**灰白**，出现「重试 / 退出至主菜单」结算项
- 全程持续卡顿，进入新场景时尤其明显

系统截图证据：`全屏_游戏画面_1109.png`、`主窗口截图_1102.png`、`主窗口截图_1102b.png`

> 说明：S2Arm 的 `snapshot` 只截主窗口区域，会把旁边的游戏窗口裁成竖窄条；全屏截图确认游戏窗口本身是横屏、完整显示。竖窄条不是问题现象。

## 日志证据（第二轮 `09a35893-2e7c-434e-9206-2ee064ca1a10-game.log`）

| 指标 | 实测值 |
|---|---|
| `[longframe]` 总数 | 1258 |
| 其中整帧 > 1000 ms | 46 |
| 最长样例 | `frame 7612 whole frame 3441.0 ms, submit+present 3426.1 ms, GPU 14.3 ms; new pipelines 9 + shader libraries 27, 5074.1 ms in all` |
| 另一例 | `frame 11343 whole frame 1089.8 ms, submit+present 1060.8 ms, GPU 11.4 ms; new pipelines 2 + shader libraries 4, 940.5 ms in all` |
| `直连着色器首遇现编` | 821 次 |
| `loaded from the fallback translation` | 1165 条 |
| 收局汇总 | `直连着色器现编汇总：成功 821、失败 0、跳过 0、复用上局 344、输出类型变体 1` |
| 后台预编 | `交后台 256 支、编好 256 支；首遇时已编好 5 支` |
| **顶点补齐变体** | **`设备编好 0 / 编不出 42`** |
| 具体失败行 | `顶点补齐变体：程序号 01006560184E6000 顶点 0811cff99be9aa3c 补 user(locn) 掩码 0x40 后台编失败 0.0 ms：the game no longer holds its control section to translate it` |
| GPU 挂死安全停 | 本轮 0 次（首轮 `a97a9207` 有 1 次） |

关键点：长帧里 GPU 只占 10–20 ms，耗时几乎全在 `submit+present` 与**运行时现场编译着色器**；GPU 不是瓶颈。

## 相关配置现状

- `image/mac/shaders/direct/shader_map.tsv` 为 **0 字节**，`keys.tsv` 为 **0 字节**，`shader_facts.tsv` 写着「运行时现编空库：空答案表 = 全部按缺省答」「空配对表」
- `image/mac/shaders/direct/game.tsv` 头部自述：`Provisional game profile (generated ... from the survey with dm/games/profile_guess.py, not checked by hand)` —— 自动猜测生成、未经人工核对
- `image/mac/s2a_caps.tsv`：`widen 0`（发布缺省关）、`fps60 1`、`direct 1`
- 日志：`宽屏：main 74 个点，启用 0 个，屏幕宽 1920`
- 日志：`采样器 LOD 偏置实测：不生效（Apple M1），走着色器模拟`
- 日志：`[S2A 画面] 渲染倍率 1x，抗锯齿 None，缩放滤镜 Bilinear`
- 日志：`原生 60 帧判定：否`

## 已排除

- 不是窗口尺寸 / 竖屏裁切问题（全屏截图已证）
- 不是出包失败（编译 `completed`，`hostFailed: []`）
- 不是 `GPU 挂死安全停` 导致本轮画面异常（本轮计数 0）
- 不是 0.2.2/0.2.3 的 `.DS_Store` 问题（版本已 0.2.6）

## 期望

需要维护方确认 MK1 的直连着色器档是否需要补齐，特别是「顶点补齐变体 42 个全编不出」是否即为角色模型不渲染的原因。如需要，可提供完整 `game.log` 与诊断包。

## 附件

- `S2Arm_MK1_诊断包.zip`（`s2armctl diag export` 导出，1.38 MB）
- `全屏_游戏画面_1109.png`、`主窗口截图_1102.png`、`主窗口截图_1102b.png`
- 原始日志 `~/S2Arm/logs/09a35893-2e7c-434e-9206-2ee064ca1a10-game.log`（3.56 MB）
