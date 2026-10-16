# Tactical Sprint — Fika 增强套件

三个给 **[FROM HELL - Tactical Sprint](https://github.com/)** 用的小 mod，**解决它在联机和平衡上的三个问题**。

| 项目 | 解决什么 | 依赖 | 装哪 |
|---|---|---|---|
| **[TacticalSprint.Fika](src/TacticalSprint.Fika)** | 战术冲刺动作在 **Fika 联机**里看不到 | Fika + Tactical Sprint | 每个客户端（含无头端） |
| **[TacticalSprint.WeightGate](src/TacticalSprint.WeightGate)** | 背 60 kg 还能战术冲刺，**破坏平衡** | Tactical Sprint | 客户端 |
| **[TacticalSprint.SpeedFix](src/TacticalSprint.SpeedFix)** | 原 mod **只让动画变快，位移没变** | 无 | 客户端 |

> 三个可以**任意组合**安装，互不冲突。

---

## 三个问题分别是什么

### ① 联机里别人看不到你的战术冲刺

`FROM HELL - Tactical Sprint` 是纯客户端 mod —— 只改**你自己的**动画状态机。
「是否处于战术冲刺」这个状态从来没被发到网络上。

⇒ **你的屏幕**：你在战术冲刺 · **别人的屏幕**：你在普通奔跑

**`TacticalSprint.Fika`** 把这个状态广播出去，让别人的客户端在观察模型上重放同样的动画。

### ② 负重不设限

原 mod 让玩家**无条件**能战术冲刺，不管背包多重。

**`TacticalSprint.WeightGate`** 在它的准入判断上挂钩子，负重超门槛就否决。
门槛可配：有任何负重就禁（约 26 kg）/ 只有过重才禁（约 77 kg）/ 不限制。

### ③ 位移没真的变快

反编译确认，原 mod 改的是 `MovementContext.SprintAcceleration` 里的一个局部变量，它最终只流向
`Animator.SetFloat(SPRINT_SPEED)` —— 而 `_sprintSpeed` 这个字段**整个程序集里只有它自己读**，
**没有任何移动代码消费它**。

⇒ **腿部动画变快了，人还是跑那么快。**

真正管位移的链是：

```
SpeedLimiter.GetSpeed(...)
  → SpeedLimiter.Speed
    → OnSpeedChanged
      → MovementContext.OnSpeedLimiterSpeedChanged
        → CharacterController.SpeedLimit = speed      ← ★ 物理层，这里才生效
```

**`TacticalSprint.SpeedFix`** 在这条链的源头做 Postfix，直接放大 `CharacterController.SpeedLimit`。

---

## 安装

**三个都需要 [BepInEx 5](https://github.com/BepInEx/BepInEx) + SPT 4.1.x。**

把需要的 dll 放进 `BepInEx\plugins\`：

```
BepInEx\plugins\
  TacticalSprint.Fika.dll           （联机同步，每个客户端 + 无头端都要）
  TacticalSprintWeightGate.dll      （负重限制）
  TacticalSprintSpeedFix.dll        （速度修正）
```

**`TacticalSprint.Fika` 的前置：**
1. **[Fika](https://github.com/project-fika/Fika-Plugin)**（Fika.Core 2.4.x）
2. **FROM HELL - Tactical Sprint**

> 无头端也需要装 `TacticalSprint.Fika` —— 它是主机，要能收发这个包。

---

## 配置

三个各有一份 cfg，都在 `BepInEx\config\`：

```
com.local.tacticalsprint.fika.cfg          无配置项，装上即生效
com.local.tacticalsprintweightgate.cfg     Enabled / Gate / Debug
com.local.tacticalsprintspeedfix.cfg       Enabled / LocalPlayerOnly / Debug / SprintMultiplier
```

**关键项：**

| mod | 项 | 默认 | 说明 |
|---|---|---|---|
| WeightGate | `Gate` | `Encumbered` | `Encumbered`（约26kg起禁）/ `OverEncumbered`（约77kg起禁）/ `None` |
| SpeedFix | `SprintMultiplier` | `2.0` | 1~10。**默认 2 倍提升很大，想温和先试 1.3~1.5** |
| SpeedFix | `LocalPlayerOnly` | `true` | 只对本地玩家生效（推荐）。关闭会连 AI 一起加速 |

> ⚠ **SpeedFix 的倍率建议联机里所有人用同一个值**，否则移动速度观感不一致。

---

## 构建

```powershell
# 编全部三个
dotnet build TacticalSprint-Companions.sln -c Release -p:EFT_DIR="你的 SPT 安装目录"

# 或单独编一个
cd src\TacticalSprint.SpeedFix
dotnet build -c Release -p:EFT_DIR="D:\MySPT"
```

`EFT_DIR` 是 csproj 里 `$(EFT_DIR)` 的默认值，默认 `C:\SPT`。它必须指向**含 `EscapeFromTarkov.exe` 和 `BepInEx` 的目录**；指错了会立刻报一条清楚的错误。

`TacticalSprint.Fika` 额外从这些位置取引用（都在 `$(EFT_DIR)` 下）：
```
BepInEx\plugins\Fika\Fika.Core.dll
BepInEx\plugins\FromHell.TacticalSprint\FromHell.TacticalSprint.dll
```

产物在各自 `src\<项目>\bin\Release\`。

---

## 已知问题

- **三个都依赖原 mod 的内部 API** —— FROM HELL - Tactical Sprint 大改版可能要跟着适配
- **`TacticalSprint.Fika` 不同步速度** —— 只同步「是否在冲刺」这个布尔
- **`TacticalSprint.SpeedFix` 是验证版 v0.1.0** —— 逻辑基于反编译推导，实机验证有限
- **`SpeedFix` 和原 mod 同时开会叠加**（原 mod 改动画，SpeedFix 改位移）
- **中途加入的玩家**：要等下一次状态变化才会看到你的战术冲刺

---

## 许可

MIT