# Tactical Sprint — 负重限制

让 **[FROM HELL - Tactical Sprint](https://github.com/)** 尊重负重 —— **背太重就冲不起来**。

> **依赖：** FROM HELL - Tactical Sprint（软依赖：没装就静默不生效，不报错）

---

## 解决什么问题

原版 Tactical Sprint 让玩家**无条件**能战术冲刺，不管背包有多重。

⇒ 背着 60 kg 还能战术冲刺，破坏平衡。

**本插件在它的准入判断上挂钩子**，负重超门槛就否决。

---

## 安装

1. 先装 **FROM HELL - Tactical Sprint**
2. 把 `TacticalSprintWeightGate.dll` 放进 `BepInEx\plugins\`

日志确认：

```
[TacticalSprintWeightGate] v1.0.0 已加载。patching=4/4, Enabled=True, Gate=Encumbered, Debug=False
```

---

## 配置

配置文件：`BepInEx\config\com.local.tacticalsprintweightgate.cfg`

### `[General]`

| 项 | 默认 | 说明 |
|---|---|---|
| `Enabled` | `true` | 总开关。关掉后**完全不干预** TacticalSprint |
| `Gate` | `Encumbered` | **负重门槛**（见下表） |
| `Debug` | `false` | 调试日志（绑定/解绑、状态变化、兜底纠正、拦截与放行）。**只在状态变化时输出，不刷屏** |

### `Gate` 的三个取值

| 值 | 含义 | 大约重量 |
|---|---|---|
| **`Encumbered`** | 有任何负重就禁止（默认） | **约 26 kg 起** |
| **`OverEncumbered`** | 只有过重才禁止 | **约 77 kg 起** |
| **`None`** | 不限制（等于禁用本插件） | — |

> **门槛基于玩家自己的负重上限**，已经含力量技能与健康状态修正 —— 不是硬编码的公斤数。
> 上表的公斤数只是"一个典型角色"的参考值。

---

## 工作原理

### 4 个补丁

| # | 目标 | 作用 |
|---|---|---|
| 1 | `FromHell.TacticalSprint.Plugin.Eligible(Player)` | **★ 核心** —— Postfix 改写准入判定 |
| 2 | `Plugin.Bind(Player)` | 跟着原 mod 的生命周期绑定负重事件 |
| 3 | `Plugin.Clear()` | 解绑 |
| 4 | `Plugin.OnDestroy()` | 释放 |

> 2~4 只是**让事件订阅跟着原 mod 的生命周期走**。找不到也不致命 —— 会退化成按需重绑。

### 负重事件

订阅 `PhysicalBase` 的：

```
EncumberedChanged
OverEncumberedChanged
```

**⚠ 这两个事件是边沿触发的** —— 所以订阅前必须**先同步一次当前值**，
否则「订阅时就已经处于负重状态」会永远收不到通知。代码里显式处理了这点。

### 兜底

除了挂钩子，还有一个周期性检查：如果负重状态已变但冲刺标志还挂着，就主动纠正。

---

## 已知问题

- **依赖原 mod 的内部 API**（`Plugin.Eligible` / `Bind` / `Clear` / `OnDestroy`）—— 原 mod 大改版可能要跟着适配
- **只对本地玩家生效**：不去干预别人的判断
- **AI 不受限**：本插件只处理玩家

---

## 构建

```powershell
dotnet build -c Release -p:EFT_DIR="你的 SPT 安装目录"
```

`EFT_DIR` 默认 `C:\SPT`。产物在 `bin\Release\TacticalSprintWeightGate.dll`。

> 用 `AccessTools.TypeByName("FromHell.TacticalSprint.Plugin")` 做**运行时**查找，
> 所以编译时**不需要**引用原 mod 的 dll。

---

## 许可

MIT