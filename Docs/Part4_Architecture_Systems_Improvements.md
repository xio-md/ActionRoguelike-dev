# Part 4: ActionRoguelike 架构模式与系统设计改进方案

> 基于 PX 项目的 C++/TypeScript 双语言桥接、GlobalMessage 事件总线、SDK 封装模式、技能系统架构的分析，提出适用于 ActionRoguelike 的架构层面改进方案。

---

## 目录

1. [事件总线架构升级](#1-事件总线架构升级)
2. [OnlineSubsystem 抽象封装模式](#2-onlinesubsystem-抽象封装模式)
3. [动作系统架构对比与优化](#3-动作系统架构对比与优化)
4. [数据驱动设计深化](#4-数据驱动设计深化)
5. [实施路线图](#5-实施路线图)

---

## 1. 事件总线架构升级

### 1.1 PX 的 GlobalMessage 设计要点

PX 的 `UGMMessageManager` 是一个进程内类型安全事件总线，其核心设计特点：

| 特性 | PX 实现 | 价值 |
|-----|--------|------|
| **类型安全** | 模板参数推导 + Editor 模式运行时签名校验 | 编译期 + 运行期双重保障 |
| **三种消息模式** | Notify（通知）、Request-Response（请求-响应）、Listen（订阅） | 覆盖全部通信场景 |
| **生命周期管理** | `WatchedObj` GC 自动解绑、`Times` 次数限制 | 防止悬空回调 |
| **嵌套调用** | `MessageBodyStack` 栈跟踪 | 调试友好 |
| **网络透明** | `URpcProxyComponent` 跨网络桥接 | 本地远程无感知 |

### 1.2 ActionRoguelike 当前状态

`URogueMessagingSubsystem` 有基础框架但缺少：
- 类型安全校验（依赖 `FInstancedStruct` 运行时匹配）
- 请求-响应模式
- 自动生命周期管理
- 执行次数限制

### 1.3 改进方案：升级为完整事件总线

#### 1.3.1 添加 Request-Response 模式

```cpp
// RogueMessagingSubsystem.h 新增

USTRUCT()
struct FRogueMessageResponse
{
    GENERATED_BODY()
    
    uint64 SequenceId;
    TFunction<void(FInstancedStruct)> Callback;
    TWeakObjectPtr<UObject> Owner;
};

// 发起异步请求
template<typename TRequest, typename TResponse>
uint64 RequestMessage(FGameplayTag MessageTag, const TRequest& Request, 
    UObject* Owner, TFunction<void(const TResponse&)> Callback)
{
    static uint64 NextSequenceId = 0;
    uint64 SeqId = ++NextSequenceId;
    
    // 存储回调
    FRogueMessageResponse Response;
    Response.SequenceId = SeqId;
    Response.Owner = Owner;
    Response.Callback = [Callback](FInstancedStruct Payload)
    {
        if (const TResponse* Resp = Payload.GetPtr<TResponse>())
        {
            Callback(*Resp);
        }
    };
    PendingResponses.Add(SeqId, MoveTemp(Response));
    
    // 发送请求
    FInstancedStruct Payload = FInstancedStruct::Make<TRequest>(Request);
    BroadcastTag(MessageTag, Payload);
    
    return SeqId;
}

// 响应请求（服务端调用）
template<typename TRequest, typename TResponse>
void RespondToMessage(uint64 RequestSequence, const TResponse& Response)
{
    // 查找对端并发送响应消息（Tag = RequestTag + _Response）
    // ...
}

// 服务器收到请求
void URogueMessagingSubsystem::OnRequestReceived(
    FGameplayTag RequestTag, FInstancedStruct Payload, APlayerController* Requester)
{
    // 分发给注册了该 Tag 的处理器
    // 处理器可以调用 RespondToMessage() 响应
}
```

#### 1.3.2 添加自动生命周期管理

```cpp
struct FRogueMessageListener
{
    TWeakObjectPtr<const UObject> WatchedObject;  // 被监听对象
    TWeakObjectPtr<UObject> OwnerObject;           // 回调所有者
    FOnMessageReceived Callback;
    int32 RemainingCalls = -1;  // -1 表示无限
    
    bool IsValid() const
    {
        return (!WatchedObject.IsValid() || WatchedObject.IsValid())  // WatchedObj 可以为空
            && OwnerObject.IsValid()
            && (RemainingCalls != 0);
    }
};

void URogueMessagingSubsystem::ListenMessage(
    FGameplayTag Tag, UObject* Owner, FOnMessageReceived Callback, 
    const UObject* WatchedObj = nullptr, int32 Times = -1)
{
    FRogueMessageListener Listener;
    Listener.WatchedObject = WatchedObj;
    Listener.OwnerObject = Owner;
    Listener.Callback = Callback;
    Listener.RemainingCalls = Times;
    
    MessageListeners.FindOrAdd(Tag).Add(MoveTemp(Listener));
}

void URogueMessagingSubsystem::BroadcastTag(FGameplayTag InTag, FInstancedStruct InPayload)
{
    TArray<FRogueMessageListener>* Listeners = MessageListeners.Find(InTag);
    if (!Listeners) return;
    
    // 逆序遍历，边广播边清理无效监听器
    for (int32 i = Listeners->Num() - 1; i >= 0; --i)
    {
        FRogueMessageListener& Listener = (*Listeners)[i];
        
        if (!Listener.IsValid())
        {
            Listeners->RemoveAt(i);
            continue;
        }
        
        Listener.Callback.ExecuteIfBound(InTag, InPayload);
        
        if (Listener.RemainingCalls > 0)
        {
            Listener.RemainingCalls--;
        }
    }
}
```

#### 1.3.3 添加 Editor 运行时签名校验（可选）

```cpp
#if WITH_EDITOR
// 在 Editor 模式下记录每个 Tag 期望的 Payload 类型
// 运行时检测到类型不匹配时输出 Error
struct FMessageSignatureInfo
{
    UScriptStruct* ExpectedPayloadType;
    FString RegisteredLocation; // 首次注册的代码位置
};

// RegisterRequestMessage<TRequest, TResponse>(Tag) 时写入签名
// BroadcastTag(Tag, Payload) 时比对签名
#endif
```

### 1.4 预期收益

| 改进项 | 收益 |
|-------|------|
| Request-Response 模式 | 异步查询（如"获取玩家状态"）无需手动管理回调 |
| GC 安全生命周期 | 防止最严重的崩溃类别（GC'd 对象回调） |
| 次数限制 | 一次性事件监听（如"等待玩家死亡"） |
| Editor 签名校验 | 早期发现 message 类型不匹配 bug |

---

## 2. OnlineSubsystem 抽象封装模式

### 2.1 PX 的 OnlineSystem 抽象层设计

PX 使用 `IOnlineSystem` 接口将第三方 SDK 拆分为三个正交接口：

```
IOnlineSystem (抽象基类)
├── IOnlineConnector  → GCloud IConnector（网络连接）
├── IOnlineAccount    → MSDK（账号鉴权）
└── IOnlineDir        → TDir（区服导航）
```

这种设计使得 SDK 替换（如 GCloud → AWS GameLift）只需实现新接口，**业务层代码零改动**。

### 2.2 ActionRoguelike 当前状态

直接使用 `IOnlineSubsystem::Get()`，与具体实现（Steam/EOS）紧耦合。如果未来想换平台（如 EOS → Steam），需要修改多处代码。

### 2.3 改进方案：轻量级抽象层

```cpp
// RogueOnlineSubsystemBase.h

/**
 * 在线系统抽象接口
 * 隔离第三方 SDK 依赖，使业务层不感知平台差异
 */
class URogueOnlineSubsystemBase : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    // === 会话 ===
    virtual void CreateSession(int32 MaxPlayers, bool bIsLAN) PURE_VIRTUAL(,)
    virtual void FindSessions() PURE_VIRTUAL(,)
    virtual void JoinSession(const FBlueprintSessionResult& Session) PURE_VIRTUAL(,)
    virtual void DestroySession() PURE_VIRTUAL(,)
    
    // === 好友 ===
    virtual void ReadFriendsList() PURE_VIRTUAL(,)
    virtual TArray<FString> GetFriendsList() const PURE_VIRTUAL(,)
    
    // === 身份 ===
    virtual FString GetPlayerNickname() const PURE_VIRTUAL(,)
    virtual FUniqueNetIdRepl GetUniquePlayerId() const PURE_VIRTUAL(,)
    
    // === 平台特性 ===
    virtual bool SupportsCrossPlatform() const { return false; }
    virtual bool SupportsPresence() const { return true; }
    
    // 工厂方法
    static TSubclassOf<URogueOnlineSubsystemBase> GetPlatformClass();
};

// === Steam 实现 ===
UCLASS()
class URogueOnlineSubsystemSteam : public URogueOnlineSubsystemBase
{
    // 使用 IOnlineSubsystem::Get()->GetSessionInterface() 实现
};

// === EOS 实现（未来） ===
UCLASS()
class URogueOnlineSubsystemEOS : public URogueOnlineSubsystemBase
{
    // 使用 EOS SDK 实现
};

// === 空实现（离线/编辑器） ===
UCLASS()
class URogueOnlineSubsystemNull : public URogueOnlineSubsystemBase
{
    // 所有操作返回空/默认值，不依赖任何 SDK
};
```

**工厂方法：**
```cpp
TSubclassOf<URogueOnlineSubsystemBase> URogueOnlineSubsystemBase::GetPlatformClass()
{
    // 根据编译宏或运行时配置选择实现
#if UE_BUILD_SHIPPING
    return URogueOnlineSubsystemSteam::StaticClass();
#else
    // 开发环境可以选择 Null 实现避免登录
    return URogueOnlineSubsystemNull::StaticClass();
#endif
}
```

### 2.4 预期收益

- 隔离第三方 SDK，降低依赖耦合
- 编辑器测试不需要 Steam 登录（Null 实现）
- 未来平台迁移（Steam → EOS → 自定义）成本最低

---

## 3. 动作系统架构对比与优化

### 3.1 架构对比

| 维度 | PX (GAS + Able + PXSkill) | ActionRoguelike (自研) |
|-----|--------------------------|----------------------|
| 基础框架 | Epic GAS (`UAbilitySystemComponent`) | 自研 `URogueActionComponent` |
| 技能定义 | `UGameplayAbility` + `UGameplayEffect` | `URogueAction` + `URogueActionEffect` |
| 属性系统 | GAS `FGameplayAttribute` | 自研 `FRogueAttribute` |
| 标签系统 | `FGameplayTag`（继承于 GAS） | `FGameplayTag`（自建） |
| 网络 | 完整的 Client/Server 预测 | 基础 Server RPC |
| 冷却 | GAS `CooldownGameplayEffect` | 自研 `CooldownUntil` 计时器 |
| 连击 | PX 自研 Combo 系统 | 不支持 |
| 槽位 | 7 槽位 + 自定义槽 | 无槽位概念 |

### 3.2 不需要迁移到 GAS

**ActionRoguelike 不需要迁移到 GAS**，原因：
1. 自研系统更轻量，适合学习项目和中小型游戏
2. GAS 的学习曲线和生产复杂度不适合 ActionRoguelike 的定位
3. ActionRoguelike 的自研系统已经具备可用的基础功能

### 3.3 可以借鉴的改进

#### 3.3.1 冷却系统优化（参考 PX SkillCD）

```cpp
// 当前：各 Action 自行管理 CooldownUntil
// 改进：统一的冷却管理器

USTRUCT()
struct FRogueCooldownInfo
{
    GENERATED_BODY()
    
    UPROPERTY()
    FGameplayTag ActionTag;
    
    UPROPERTY()
    double CooldownEndTime = 0.0;
    
    UPROPERTY()
    float BaseCooldown = 0.0f;
    
    UPROPERTY()
    float CDModifier = 1.0f;  // 可由 Buff/Debuff 修改
    
    float GetRemainingTime(double CurrentTime) const
    {
        return FMath::Max(0.0, CooldownEndTime - CurrentTime);
    }
    
    float GetRemainingPct(double CurrentTime) const
    {
        float EffectiveCD = BaseCooldown * CDModifier;
        return EffectiveCD > 0 ? GetRemainingTime(CurrentTime) / EffectiveCD : 0.f;
    }
};
```

#### 3.3.2 连击系统（可选，近战游戏才需要）

PX 的连击系统通过 `Client → Server RPC → Server 验证 → Client 回调` 实现。对于 ActionRoguelike 的近战攻击（如果未来添加），可以参考：

```cpp
// 连击输入缓冲
struct FRogueComboInput
{
    FGameplayTag ActionTag;
    double InputTime;
    int32 ComboIndex;  // 第几段连击
};

// 客户端缓冲输入，服务器验证
TArray<FRogueComboInput> ComboBuffer;  // 最多缓存 3 个输入

void URogueActionComponent::PushComboInput(FGameplayTag ActionTag)
{
    if (ComboBuffer.Num() < 3) // 上限 3 段
    {
        ComboBuffer.Add({ActionTag, FPlatformTime::Seconds(), ComboBuffer.Num()});
        ServerPushComboInput(ActionTag, ComboBuffer.Num());
    }
}
```

### 3.4 预期收益

| 改进项 | 收益 |
|-------|------|
| 统一冷却管理 | 支持 Buff/Debuff 修改 CD、UI 进度条 |
| 连击缓冲 | 更流畅的近战手感（如果实现） |
| 架构参考 | 了解商业级动作系统设计思路 |

---

## 4. 数据驱动设计深化

### 4.1 PX 的配置模式

PX 大量使用 DataAsset + INI 配置，让策划在不修改 C++ 的情况下调整数值：

```
技能配置: UGameplayAbility → FGameplayTag → DataAsset
池配置:   UPXActorPoolSettings → INI
网络配置: ReplicationGraph → CVars
```

### 4.2 ActionRoguelike 已有基础

ActionRoguelike 已经在使用：
- `URogueMonsterData` (PrimaryDataAsset) - 怪物配置
- `URogueProjectileData` (DataAsset) - 投射物配置
- `URogueDeveloperSettings` (DeveloperSettings) - 开发者配置
- `URogueSaveGameSettings` (DeveloperSettings) - 存档配置

### 4.3 扩展建议

#### 4.3.1 Actor 池配置 DataAsset

```cpp
UCLASS()
class URoguePoolingDataAsset : public UPrimaryDataAsset
{
    GENERATED_BODY()
    
public:
    UPROPERTY(EditAnywhere, Category = "Pooling")
    TMap<TSubclassOf<AActor>, int32> PrewarmCounts; // 类 -> 预热数量
    
    UPROPERTY(EditAnywhere, Category = "Pooling")
    TMap<TSubclassOf<AActor>, int32> MaxPoolSizes;  // 类 -> 最大容量
    
    UPROPERTY(EditAnywhere, Category = "Pooling")
    float GlobalIdleTimeout = 300.0f;
};
```

#### 4.3.2 AI 波次配置 DataTable

```cpp
USTRUCT(BlueprintType)
struct FRogueWaveConfig : public FTableRowBase
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere)
    TSubclassOf<ARogueAICharacter> MonsterClass;
    
    UPROPERTY(EditAnywhere)
    int32 Count = 1;
    
    UPROPERTY(EditAnywhere)
    float SpawnInterval = 1.0f;
    
    UPROPERTY(EditAnywhere)
    int32 CreditCost = 10;
};

// 在 GameMode 中从 DataTable 读取波次配置
UDataTable* WaveConfigTable;
```

#### 4.3.3 网络配置 DataAsset

```cpp
UCLASS(Config=Game, DefaultConfig)
class URogueNetworkSettings : public UDeveloperSettings
{
    GENERATED_BODY()
    
public:
    UPROPERTY(Config, EditAnywhere, Category = "Replication")
    int32 MaxPlayers = 4;
    
    UPROPERTY(Config, EditAnywhere, Category = "Replication")
    float ProjectileCullDistance = 10000.f; // 100m
    
    UPROPERTY(Config, EditAnywhere, Category = "Replication")
    float AICullDistance = 15000.f; // 150m
    
    UPROPERTY(Config, EditAnywhere, Category = "Replication")
    int32 PlayerStateReplicationFrequency = 2; // 每帧 2 个
    
    UPROPERTY(Config, EditAnywhere, Category = "Connection")
    float PingCheckInterval = 5.0f;
    
    UPROPERTY(Config, EditAnywhere, Category = "Connection")
    float ConnectionTimeout = 30.0f;
};
```

### 4.4 预期收益

- 策划/设计师不接触 C++ 即可调整游戏数值
- 不同关卡/模式使用不同配置
- 容易创建变体（如"困难模式"怪物配置）

---

## 5. 实施路线图

### 阶段 1：基础设施（1 周）

| 改进项 | 工作量 | 依赖 |
|-------|--------|------|
| 升级消息系统（Request-Response + GC 安全生命周期） | 中 | 无 |
| 数据驱动配置 DataAsset | 低 | 无 |

### 阶段 2：架构优化（1-2 周）

| 改进项 | 工作量 | 依赖 |
|-------|--------|------|
| OnlineSubsystem 抽象层 | 中 | 阶段 1 消息系统 |
| 统一冷却管理器 | 低 | 无 |

### 阶段 3：扩展（后续迭代）

| 改进项 | 工作量 | 依赖 |
|-------|--------|------|
| 连击系统（如果需要近战） | 高 | 阶段 2 冷却 |
| Editor 模式消息签名校验 | 低 | 阶段 1 消息系统 |

---

**文档版本**: 1.0  
**参考**: PX `UGMMessageManager` + `IOnlineSystem` + `UGPAbilityBase` + `UPXSkillComponent`
