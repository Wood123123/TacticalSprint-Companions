# Tactical Sprint — Fika 动画同步

让 **[FROM HELL - Tactical Sprint](https://github.com/)** 的战术冲刺动作在 **Fika 联机**里对其他玩家可见。

> **依赖：** Fika.Core · FROM HELL - Tactical Sprint（两者都是硬依赖）

---

## 解决什么问题

`FROM HELL - Tactical Sprint` 是纯客户端 mod —— 它只改**你自己的**动画状态机。
在 Fika 联机里：

- 你的屏幕上：你在战术冲刺 ✅
- 别人的屏幕上：你在**普通奔跑** ❌

⇒ 因为"是否处于战术冲刺"这个状态**从来没被发到网络上**。

**本插件把这个状态广播出去**，让其他玩家的客户端在观察模型上重放同样的动画。

---

## 安装

**装在每个客户端**（含无头端）：

1. 先装 **[Fika](https://github.com/project-fika/Fika-Plugin)**
2. 再装 **FROM HELL - Tactical Sprint**
3. 把 `TacticalSprint.Fika.dll` 放进 `BepInEx\plugins\`

日志确认：

```
[TacticalSprint.Fika] v1.0.2 已加载，战术冲刺动画同步已启用。
```

> **无头端也要装** —— 无头端是主机，它需要能收发这个包。

---

## 配置

**无配置项。** 装上即生效。

---

## 工作原理

参照 `tarkin.ladders.fika` 的同步套路：

```
① 注册包
   FikaEventDispatcher.SubscribeEvent<FikaNetworkManagerCreatedEvent>
     → manager.RegisterPacket<TacticalSprintPacket>(OnPacketReceived)

② 发送
   订阅原 mod 的 Plugin.OnTacticalSprintChanged 事件
     → 状态一变就发 { PlayerId = 我的 NetId, Active = true/false }
     → DeliveryMethod.ReliableOrdered

③ 接收
   收到包 → 按 PlayerId 找到对应的 observed 玩家
          → 挂上 ObservedPlayerTacticalSprintController
          → 由它写观察模型的动画参数
```

### 网络包

```csharp
public struct TacticalSprintPacket : INetSerializable
{
    public int  PlayerId;   // 谁的 NetId
    public bool Active;     // 是否处于战术冲刺
}
```

**只在状态变化时发包**（边沿触发），不是每帧 —— 网络开销可以忽略。

---

## 已知问题

- **原 mod 更新后可能要重新适配** —— 依赖它的 `Plugin.OnTacticalSprintChanged` 事件签名
- **不发送速度** —— 只同步「是否在冲刺」这个布尔。速度差异如果很大，观察模型可能看起来有点滑
- **掉包恢复**：用 `ReliableOrdered`，正常不会丢；但玩家中途加入（late join）时新玩家的状态要等下一次变化才同步

---

## 构建

```powershell
dotnet build -c Release -p:EFT_DIR="你的 SPT 安装目录"
```

`EFT_DIR` 默认 `C:\SPT`。产物在 `bin\Release\TacticalSprint.Fika.dll`。

**引用了这两个第三方 dll（从你的 SPT 安装里取）：**
```
$(EFT_DIR)\BepInEx\plugins\Fika\Fika.Core.dll
$(EFT_DIR)\BepInEx\plugins\FromHell.TacticalSprint\FromHell.TacticalSprint.dll
```

---

## 许可

MIT