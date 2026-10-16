# Tactical Sprint — 速度修正

修复 **[FROM HELL - Tactical Sprint](https://github.com/)** 的一个问题：
**它只让腿部动画变快，实际位移几乎没变。**

> **依赖：** 无（独立工作，不引用原 mod 的任何类型）
> **状态：** 验证版 v0.1.0

---

## 解决什么问题

### 原 mod 改错了杠杆

反编译确认，原 mod 改的是 `MovementContext.SprintAcceleration` 里的一个局部变量，它最终只流向：

```
set_SprintSpeed
  → PlayerAnimator.SetCharacterSprintSpeed
    → Animator.SetFloat(SPRINT_SPEED)
```

而 `_sprintSpeed` 这个字段**整个程序集里只有 `SprintAcceleration` 自己读**（外加一个调试快照）——
**没有任何移动代码消费它。**

⇒ **结果：腿部动画变快了，人还是跑那么快。**

### 真正管位移的链

```
SpeedLimiter.GetSpeed(EPlayerState, 力量等级, walkSpeedLimit)
  → SpeedLimiter.Speed
    → OnSpeedChanged 事件
      → MovementContext.OnSpeedLimiterSpeedChanged
        → _player.CharacterController.SpeedLimit = speed      ← ★ 物理层，这里才真的生效
```

**本插件在这条链的源头做 Postfix**，直接放大 `CharacterController.SpeedLimit`。

---

## 安装

1. **不需要**装 FROM HELL - Tactical Sprint（本插件独立工作）
2. 把 `TacticalSprintSpeedFix.dll` 放进 `BepInEx\plugins\`

日志确认：

```
[TacticalSprintSpeedFix] v0.1.0 已加载。patches=2/2, Enabled=True, SprintMultiplier=2x, LocalPlayerOnly=True
```

---

## 配置

配置文件：`BepInEx\config\com.local.tacticalsprintspeedfix.cfg`

### `[General]`

| 项 | 默认 | 说明 |
|---|---|---|
| `Enabled` | `true` | 总开关 |
| `LocalPlayerOnly` | `true` | **只对本地玩家的冲刺生效（推荐）**。关闭会连 AI / 观察模型一起加速 |
| `Debug` | `false` | 输出每次 Sprint 限速的变化（**仅在数值真的变了时输出**，不刷屏） |

### `[Movement]`

| 项 | 默认 | 范围 | 说明 |
|---|---|---|---|
| `SprintMultiplier` | **`2.0`** | **1 ~ 10** | 冲刺速度倍率。直接放大 `SpeedLimiter` 算出的 `CharacterController.SpeedLimit` |

> **⚠ 默认 2 倍是很大的提升。** 想要温和一点先试 1.3 ~ 1.5。
> 建议**在联机里所有人用同一个值**，否则你和别人的移动速度观感会不一致。

---

## 工作原理

### 补丁 ① — `SpeedLimiter.GetSpeed`（Postfix）

只对**本地玩家**的 Sprint 状态放大返回值：

```
if (Enabled && IsSprintState && IsLocalContext(limiter))
    __result *= SprintMultiplier;
```

「是不是本地玩家」靠 `MovementContext.CreateSpeedLimiter` 的 Postfix 建立映射：

```csharp
// SpeedLimiter 实例 → 它所属的 MovementContext
ConditionalWeakTable<SpeedLimiter, MovementContext> Owner
```

> 用 `ConditionalWeakTable` 是为了**不阻止 SpeedLimiter 被 GC** —— 普通 Dictionary 会内存泄漏。

### 补丁 ② — `MovementContext.CreateSpeedLimiter`（Postfix）

建立上面那个映射。找不到就**退化为不加门控**（会连 AI 一起加速，日志里会警告）。

---

## 已知问题

- **v0.1.0 是验证版** —— 逻辑基于反编译推导，实机验证有限
- **人机一致性**：如果只给部分玩家开，速度观感会不一致
- **可能影响体力消耗 / 脚步音量** —— 这些系统按速度计算，速度变了它们也跟着变
- **不改变原 mod**：本插件和原 mod 可以共存，但**同时开会叠加**（原 mod 改动画，本插件改位移）

---

## 构建

```powershell
dotnet build -c Release -p:EFT_DIR="你的 SPT 安装目录"
```

`EFT_DIR` 默认 `C:\SPT`。产物在 `bin\Release\TacticalSprintSpeedFix.dll`。

---

## 许可

MIT