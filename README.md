[English](README.en.md)

# SneakFall 轻功

> 坠落时按住潜行，减免摔落伤害。
> NeoForge 1.21.1 · Minecraft 1.21.1 · 开源模组（GPL-3.0）

SneakFall（轻功）是 NeoForge 模组。玩家从高处往下掉的时候按住潜行键，摔落伤害就按配置里的规则减免。规则按坠落高度分成若干阶段，每个阶段自己定触发方式和减免算法。

## 是什么

### 触发模式（mode）

| 模式 | 含义 |
| --- | --- |
| `SUSTAIN` | 落地那一刻按着潜行就减免 |
| `QTE` | 按下潜行后要在时间窗口里落地，窗口外落地不算 |

QTE 窗口按秒配置，运行时 1 秒 = 20 游戏刻。松开潜行，计时清零重来。

### 减免计算方式（reductionMode）

| 模式 | 含义 |
| --- | --- |
| `DISTANCE` | 算伤害前先从坠落距离里扣掉 N 格 |
| `PERCENT` | 按比例缩放伤害：最终 = 原始 × (1 − 比例)，超过 1 按 1 算 |

### 高度区间与保底

- 区间是半开区间 `[min, max)`，可以写 `"3.0~20.0"`、`"10~"`（没有上界）、`"~10"`（没有下界）、`"all"`（所有高度）。
- 阶段按顺序往下比，命中第一个就停。
- 一个都没命中（包括区间之间的空隙）时，用 `general` 分组里的保底（fallback）配置。默认保底不减免（`fallbackReductionValue = 0.0`），免得玩家莫名其妙摔不死。
- 某一行写坏了只跳过那一个阶段，服务器照常启动。

### 默认内置阶段

| 高度区间 | 模式 | QTE 窗口 | 减免方式 | 减免值 |
| --- | --- | --- | --- | --- |
| `3.0~20.0` | QTE | 0.4 s | DISTANCE | 1.5 格 |

## 怎么用

### 配置文件

首次启动会生成 `config/sneakfall-common.toml`，注释是中文的。配置在每次事件触发时重新读取，改完用 `/reload`（NeoForge）就生效，重启游戏也一样。

默认内容：

```toml
#SneakFall（潜行摔落减免）总体说明。
#按坠落高度区间定义减免规则：每个区间为一个阶段。
#每个阶段可独立选择触发模式（SUSTAIN / QTE）与减免计算方式（DISTANCE / PERCENT）。
#若坠落高度未命中任何阶段，则使用下方 general 分组中的保底（fallback）配置。
[general]
	#保底措施：坠落高度未落入任何 [stages] 阶段区间时使用（含区间空隙）。
	#默认不减免（0.0），保证不意外免疫。
	#Allowed Values: SUSTAIN, QTE
	fallbackMode = "SUSTAIN"
	# Default: 0.4
	# Range: 0.0 ~ 60.0
	fallbackQteWindow = 0.4
	#Allowed Values: DISTANCE, PERCENT
	fallbackReductionMode = "DISTANCE"
	# Default: 0.0
	# Range: 0.0 ~ 100.0
	fallbackReductionValue = 0.0

[stages]
	#摔落减免阶段列表（可删改）。
	#因底层配置库不支持嵌套对象数组（[[stages]] 数组表），改为 5 个等长平行列表：
	#第 i 个阶段由 heightRanges[i]、modes[i]、qteWindows[i]、reductionModes[i]、reductionValues[i] 组合而成。
	#示例（第 0 个阶段）：heightRanges=["3.0~20.0"]、modes=["QTE"]、qteWindows=[0.4]、
	#reductionModes=["DISTANCE"]、reductionValues=[1.5]，含义与原 [[stages]] 数组示例完全一致。
	#heightRanges 格式："min~max" 半开区间 [min,max)；"min~" 无上界；"~max" 无下界；"all" 全高度。
	#modes 取值：SUSTAIN / QTE（大小写不敏感，非法值跳过该阶段）。
	#qteWindows 仅 modes[i]=QTE 时生效，松开潜行立即归零重新计时。
	#reductionModes 取值：DISTANCE / PERCENT（大小写不敏感，非法值跳过该阶段）。
	#reductionValues：DISTANCE 模式为格数(0~100)；PERCENT 模式为比例(0~1，超 1 按 1 计)。
	#五个列表长度不一致时按最短长度截断；某一行解析失败时仅跳过该阶段。
	heightRanges = ["3.0~20.0"]
	modes = ["QTE"]
	qteWindows = [0.4]
	reductionModes = ["DISTANCE"]
	reductionValues = [1.5]
```

### 构建

```bash
./gradlew build        # Linux / macOS
gradlew.bat build      # Windows
```

构建产物在 `build/libs/`（`sneakfall-<版本>.jar`），丢进 Minecraft 实例的 `mods/` 目录就能用。

> 国内下载 Gradle 慢或者超时的话，把 `gradle/wrapper/gradle-wrapper.properties` 里的
> `distributionUrl` 换成镜像源，比如腾讯云 `https://mirrors.cloud.tencent.com/gradle/...`。

## 目录

开发过程中的文档都在 `docs/`：

| 文档 | 内容 |
| --- | --- |
| `DESIGN_SPEC.md` | 总体设计说明：目标、机制设计、配置方案 |
| `SKELETON_REPORT.md` | 项目骨架搭建记录（构建脚本、mods.toml） |
| `MECHANICS_REPORT.md` | 摔落伤害机制与同类模组调研 |
| `API_REPORT.md` | 所用 NeoForge/事件 API 签名验证记录 |
| `BUILD_REPORT.md` | 编译与验证记录（JDK、Gradle、jar 输出） |
| `CONFIG_FIX_REPORT.md` | 配置序列化崩溃修复记录（平行列表方案、保底机制） |

```
sneakfall/
├── build.gradle            # 构建脚本（NeoForge 1.21.1）
├── gradle.properties       # Gradle / 模组属性（mod_id、版本）
├── settings.gradle         # 工程设置
├── gradlew / gradlew.bat   # Gradle Wrapper 启动脚本
├── gradle/wrapper/         # Wrapper（gradle-wrapper.jar + properties）
├── src/
│   ├── main/java/me/shinano/sneakfall/
│   │   ├── SneakFall.java           # 模组入口与事件注册
│   │   ├── SneakFallConfig.java     # 配置定义 / 阶段解析 / 保底
│   │   ├── SneakFallTracker.java    # 坠落距离与潜行状态追踪
│   │   └── FallDamageHandler.java   # 摔落伤害减免处理
│   └── main/resources/META-INF/
│       └── neoforge.mods.toml       # 模组元数据（license: GPL-3.0）
├── docs/                   # 开发文档
├── README.md               # 本文件（中文）
├── README.en.md            # 英文版
└── .gitignore
```

## 依赖

- JDK 21（编译）
- Gradle 8.x：仓库自带 wrapper，不用另装
- NeoForge 21.1+ / Minecraft 1.21.1（运行）

## 边界

- 阶段配置是五条等长平行列表，用不了 `[[stages]]` 数组表，因为底层配置库不支持嵌套对象数组。五个列表长短不一时按最短的截断。
- 某一行解析失败只跳过该阶段，不拦服务器启动。反过来，写错了不会当场报错，得翻日志。
- QTE 窗口从按下潜行起算，松开归零。连续点按会一直重置计时。
- 保底默认不减免。什么都不做，好过意外把摔落伤害抹平。

## 许可

本模组按 GPL-3.0 发布，全文在仓库根目录的 `LICENSE` 里。
