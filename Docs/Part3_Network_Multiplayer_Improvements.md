# Part 3: ActionRoguelike 网络多人游戏改进方案

> 基于 PX 项目的 ReplicationGraph 空间化复制、GlobalMessage 事件总线、连接管理、SDK 封装等技术的分析，提出适用于 ActionRoguelike 的网络层改进方案。

---

## 目录

1. [ReplicationGraph 空间化复制](#1-replicationgraph-空间化复制)
2. [GlobalMessage 网络化事件总线](#2-globalmessage-网络化事件总线)
3. [连接监控与心跳](#3-连接监控与心跳)
4. [OnlineSubsystem 会话管理](#4-onlinesubsystem-会话管理)
5. [多人游戏优化细节](#5-多人游戏优化细节)
6. [实施路线图](#6-实施路线图)

---

## 1. ReplicationGraph 空间化复制

### 1.1 当前状态

ActionRoguelike 使用 UE 默认的逐 Actor `IsNetRelevantFor()` 判断复制。在多 AI 敌人 + 投射物场景中，这个 O(N×M) 判断会产生不必要的网络开销。

**问题**：
- 数十个 AI 敌人对所有玩家逐个检查可见性
- 投射物虽短生命周期但每帧都被检查
- PlayerState 没有频率限制
- 没有空间预分组加速

### 1.2 改进方案

参考 PX `UPXReplicationGraph` 实现轻量级 ReplicationGraph：

#### 1.2.1 节点架构

```cpp
// RogueReplicationGraph.h

UCLASS()
class URogueReplicationGraph : public UReplicationGraph
{
    GENERATED_BODY()
    
public:
    virtual void InitGlobalActorClassSettings() override;
    virtual void InitGlobalGraphNodes() override;
    virtual void RouteAddNetworkActorToNodes(
        const FNewReplicatedActorInfo& ActorInfo, 
        FGlobalActorReplicationInfo& GlobalInfo) override;
    
private:
    // 空间网格节点（动态 Actor）
    UPROPERTY()
    TObjectPtr<UReplicationGraphNode_GridSpatialization2D> GridNode;
    
    // 始终可见节点（GameState, GameMode 等）
    UPROPERTY()
    TObjectPtr<UReplicationGraphNode_ActorList> AlwaysRelevantNode;
    
    // PlayerState 频率限制器
    UPROPERTY()
    TObjectPtr<UReplicationGraphNode_PlayerStateFrequencyLimiter> PlayerStateNode;
};
```

#### 1.2.2 Actor 路由策略

```cpp
void URogueReplicationGraph::InitGlobalActorClassSettings()
{
    Super::InitGlobalActorClassSettings();
    
    // 显式路由表
    auto AddRouting = [this](UClass* Class, EClassRepNodeMapping Policy)
    {
        FClassReplicationInfo& Info = GlobalActorReplicationInfoMap.Get(Class);
        switch (Policy)
        {
        case EClassRepNodeMapping::AlwaysRelevant:
            Info.SetCullDistanceSquared(0.f);
            break;
        case EClassRepNodeMapping::Spatialize_Dynamic:
            Info.SetCullDistanceSquared(FMath::Square(15000.f)); // 150m
            Info.SetReplicationPeriodFrame(3); // 每 3 帧复制一次
            break;
        case EClassRepNodeMapping::Spatialize_Static:
            Info.SetCullDistanceSquared(FMath::Square(20000.f)); // 200m
            break;
        }
    };
    
    // GameState：始终可见
    AddRouting(ARogueGameState::StaticClass(), EClassRepNodeMapping::AlwaysRelevant);
    
    // AI 角色：空间化动态（有剔除距离）
    AddRouting(ARogueAICharacter::StaticClass(), EClassRepNodeMapping::Spatialize_Dynamic);
    
    // 玩家角色：空间化动态（大距离，基本不剔除）
    auto* PlayerInfo = &GlobalActorReplicationInfoMap.Get(ARoguePlayerCharacter::StaticClass());
    PlayerInfo->SetCullDistanceSquared(FMath::Square(50000.f)); // 500m
    
    // 投射物：空间化动态（短距离）
    auto* ProjectileInfo = &GlobalActorReplicationInfoMap.Get(ARogueProjectile::StaticClass());
    ProjectileInfo->SetCullDistanceSquared(FMath::Square(10000.f)); // 100m
    ProjectileInfo->SetReplicationPeriodFrame(1); // 每帧（移动快，需要频繁更新）
    
    // 拾取物：空间化静态（不移动，大距离）
    AddRouting(ARoguePickupActor::StaticClass(), EClassRepNodeMapping::Spatialize_Static);
}
```

#### 1.2.3 初始化全局图节点

```cpp
void URogueReplicationGraph::InitGlobalGraphNodes()
{
    Super::InitGlobalGraphNodes();
    
    // 空间网格：100m 网格单元（PX 默认值）
    GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
    GridNode->CellSize = 10000.f; // 100m
    
    // 始终可见节点
    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_ActorList>();
    
    // PlayerState 频率限制：每帧最多 2 个
    PlayerStateNode = CreateNewNode<UReplicationGraphNode_PlayerStateFrequencyLimiter>();
    PlayerStateNode->TargetFrameNumForReplication = 2;
    
    AddGlobalGraphNode(GridNode);
    AddGlobalGraphNode(AlwaysRelevantNode);
    AddGlobalGraphNode(PlayerStateNode);
}

void URogueReplicationGraph::RouteAddNetworkActorToNodes(
    const FNewReplicatedActorInfo& ActorInfo,
    FGlobalActorReplicationInfo& GlobalInfo)
{
    if (ActorInfo.Class == ARogueGameState::StaticClass() ||
        ActorInfo.Class == AGameModeBase::StaticClass())
    {
        AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
    }
    else if (ActorInfo.Class == APlayerState::StaticClass())
    {
        PlayerStateNode->NotifyAddNetworkActor(ActorInfo);
    }
    else if (ActorInfo.Class->IsChildOf(AActor::StaticClass()) && 
             !ActorInfo.Class->HasAnyClassFlags(CLASS_Abstract))
    {
        GridNode->AddActor_Dynamic(ActorInfo, GlobalInfo);
    }
}
```

#### 1.2.4 在 GameMode 中启用

```cpp
// ARogueGameModeBase.cpp

void ARogueGameModeBase::InitGame(const FString& MapName, 
    const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);
    
    // 创建自定义 ReplicationGraph（仅 DS 或 Listen Server）
    if (GetNetMode() != NM_Client)
    {
        URogueReplicationGraph* RepGraph = NewObject<URogueReplicationGraph>(this);
        GetWorld()->GetSubsystem<UReplicationDriverSubsystem>()->
            SetReplicationDriver(RepGraph);
    }
}
```

### 1.3 预期性能收益

| 场景 | 默认复制 | ReplicationGraph | 改善 |
|-----|---------|-----------------|------|
| 50 AI + 4 玩家网络开销 | O(50×4) = 200 次/帧 | O(GridCell) ≈ 20-40 次/帧 | **5-10x** |
| PlayerState 带宽占用 | 每帧所有 PS | 每帧 2 个 | **80-90%** 减少 |
| 远处 AI 复制 CPU | 全部检查 | 距离剔除跳过 | **60-80%** 减少 |
| 投射物带宽 | 全部复制 | 100m 外剔除 | **30-50%** 减少 |

**注意：** ReplicationGraph 本身不减少带宽，而是减少不必要的 CPU 复制度检查。真正的带宽节省来自：距离剔除 + 频率限制。

---

## 2. GlobalMessage 网络化事件总线

### 2.1 当前状态

ActionRoguelike 已有 `URogueMessagingSubsystem`（`UGameInstanceSubsystem`），但它仅支持本地消息传递，没有网络桥接。

**问题**：
- `Message.MonsterKilled` 只在服务器触发，客户端收不到
- 没有 `RemoveTagListener` 实现
- 没有延迟消息队列

### 2.2 改进方案

参考 PX `UGMMessageManager` + `URpcProxyComponent` 的双层设计：

#### 2.2.1 增强现有 Messaging Subsystem

```cpp
// RogueMessagingSubsystem.h 增强

UENUM(BlueprintType)
enum class ERogueMessageScope : uint8
{
    Local,          // 仅本机
    Server,         // 服务器 => 所有客户端
    Client,         // 客户端 => 服务器
    NetMulticast    // 所有人（含发送者）
};

UCLASS()
class URogueMessagingSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    // === 现有 API（保持兼容） ===
    void AddTagListener(FGameplayTag InTag, FOnMessageReceived InEventHook);
    void BroadcastTag(FGameplayTag InTag, FInstancedStruct InPayload);
    
    // === 新增：网络感知 API ===
    void BroadcastTagNet(FGameplayTag InTag, FInstancedStruct InPayload, 
        ERogueMessageScope Scope = ERogueMessageScope::Server);
    void RemoveTagListener(FGameplayTag InTag, UObject* BoundObject);
    
    // === 新增：延迟消息 ===
    void QueueTagForLateJoiners(FGameplayTag InTag, FInstancedStruct InPayload);
    void FlushQueuedMessagesForPlayer(APlayerController* NewPlayer);
    
protected:
    // === 新增：网络复制 ===
    // 通过 PlayerController 的 RPC 实现跨网络转发
    
    // 服务器收到客户端消息
    void OnClientMessageReceived(APlayerController* Sender, 
        FGameplayTag InTag, const TArray<uint8>& Payload);
    
    // 复制到客户端的消息结构
    UPROPERTY(ReplicatedUsing = OnRep_QueuedMessages)
    TArray<FRogueNetworkMessage> QueuedMessages;
    
    UFUNCTION()
    void OnRep_QueuedMessages();
    
    // 延迟消息队列（未包含在某客户端连接之前）
    TArray<FRogueNetworkMessage> PendingLateJoinMessages;
};
```

#### 2.2.2 网络桥接实现

```cpp
// 服务器广播到所有客户端
void URogueMessagingSubsystem::BroadcastTagNet(
    FGameplayTag InTag, FInstancedStruct InPayload, ERogueMessageScope Scope)
{
    switch (Scope)
    {
    case ERogueMessageScope::Local:
        BroadcastTag(InTag, InPayload); // 本地直接触发
        break;
        
    case ERogueMessageScope::Server:
        // 客户端 -> 服务器：通过 PlayerController RPC
        if (APlayerController* PC = GetFirstLocalPlayerController())
        {
            ServerSendMessage(PC, InTag, PayloadBytes);
        }
        break;
        
    case ERogueMessageScope::Client:
        // 服务器 -> 所有客户端：通过复制属性
        if (GetWorld()->GetNetMode() != NM_Client)
        {
            FRogueNetworkMessage NetMsg(InTag, PayloadBytes);
            QueuedMessages.Add(NetMsg);
            // 下一帧复制
            MarkPropertyDirty(FindFProperty<FProperty>(GetClass(), TEXT("QueuedMessages")));
        }
        break;
    }
}

// 客户端收到复制消息
void URogueMessagingSubsystem::OnRep_QueuedMessages()
{
    for (const FRogueNetworkMessage& NetMsg : QueuedMessages)
    {
        FInstancedStruct Payload;
        FMemoryReader Reader(NetMsg.PayloadData);
        Payload.Serialize(Reader);
        
        BroadcastTag(NetMsg.MessageTag, Payload);
    }
}

// 服务器收到客户端上行的消息
void URogueMessagingSubsystem::ServerSendMessage_Implementation(
    APlayerController* Sender, FGameplayTag InTag, 
    const TArray<uint8>& PayloadBytes)
{
    FInstancedStruct Payload;
    FMemoryReader Reader(PayloadBytes);
    Payload.Serialize(Reader);
    
    // 本地触发
    BroadcastTag(InTag, Payload);
    
    // 转发给其他客户端（如果需要）
    // ...
}
```

#### 2.2.3 使用示例

```cpp
// 服务器：AI 被击杀，通知所有客户端
FInstancedStruct Payload = FInstancedStruct::Make<FMonsterKilledPayload>();
Payload.Get<FMonsterKilledPayload>().VictimLocation = KilledAI->GetActorLocation();
Payload.Get<FMonsterKilledPayload>().KillerController = Killer;

GetGameInstance()->GetSubsystem<URogueMessagingSubsystem>()->
    BroadcastTagNet(
        SharedGameplayTags::Message_MonsterKilled, 
        Payload, 
        ERogueMessageScope::Client);

// 客户端：监听并播放击杀特效/音效/UI
MessagingSubsystem->AddTagListener(
    SharedGameplayTags::Message_MonsterKilled,
    FOnMessageReceived::CreateLambda([](FGameplayTag Tag, FInstancedStruct Payload)
    {
        // 播放击杀反馈
    }));
```

#### 2.2.4 扩展 GameplayTag 消息命名空间

```cpp
// SharedGameplayTags.h 新增
UE_DECLARE_GAMEPLAY_TAG_EXTERN(Message_PlayerJoined);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(Message_PlayerLeft);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(Message_GameStarted);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(Message_GameEnded);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(Message_PlayerDied);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(Message_MonsterSpawned);
UE_DECLARE_GAMEPLAY_TAG_EXTERN(Message_PowerupCollected);
```

### 2.3 预期收益

| 改进点 | 收益 |
|-------|------|
| 网络化消息 | UI/音效/VFX 客户端正确响应服务端事件 |
| RemoveTagListener | 消除 GC 崩溃风险 |
| 延迟消息队列 | 后加入玩家正确收到之前的状态通知 |
| 消息 Tag 扩展 | 完整的事件系统覆盖 |

---

## 3. 连接监控与心跳

### 3.1 当前状态

ActionRoguelike 依赖 UE 默认的 NetDriver 超时机制（`ConnectionTimeout` = 默认 60s）。没有自定义心跳、没有连接状态监控、没有重连机制。

### 3.2 改进方案

参考 PX `UCSNetworkSystem` 的轻量级心跳设计：

```cpp
// RogueConnectionMonitor.h

UCLASS()
class URogueConnectionMonitor : public UWorldSubsystem, public FTickableGameObject
{
    GENERATED_BODY()
    
public:
    virtual void Tick(float DeltaTime) override;
    virtual TStatId GetStatId() const override;
    virtual bool IsTickable() const override;
    
    DECLARE_MULTICAST_DELEGATE_TwoParams(FOnConnectionQualityChanged, 
        APlayerController*, float /*PingMs*/);
    FOnConnectionQualityChanged OnConnectionQualityChanged;
    
    DECLARE_MULTICAST_DELEGATE_OneParam(FOnPlayerTimedOut, 
        APlayerController*);
    FOnPlayerTimedOut OnPlayerTimedOut;
    
private:
    struct FConnectionInfo
    {
        TWeakObjectPtr<APlayerController> PlayerController;
        float LastPingTime = 0.f;
        float AvgPingMs = 0.f;
        int32 ConsecutiveTimeouts = 0;
    };
    
    TArray<FConnectionInfo> Connections;
    
    static constexpr float PingInterval = 5.0f;   // 5 秒一次
    static constexpr float TimeoutThreshold = 30.0f; // 30 秒无响应
    static constexpr int32 MaxConsecutiveTimeouts = 3;
    
    void UpdatePlayerPing(APlayerController* PC);
};

// Tick 实现
void URogueConnectionMonitor::Tick(float DeltaTime)
{
    if (GetWorld()->GetNetMode() == NM_Client) return;
    
    for (auto& Conn : Connections)
    {
        APlayerController* PC = Conn.PlayerController.Get();
        if (!PC) continue;
        
        // 周期检查（5 秒）
        if (FPlatformTime::Seconds() - Conn.LastPingTime > PingInterval)
        {
            Conn.LastPingTime = FPlatformTime::Seconds();
            
            // 通过 UE 内置 Ping 获取延迟
            float PingMs = PC->PlayerState ? 
                PC->PlayerState->GetPingInMilliseconds() : 0.f;
            
            Conn.AvgPingMs = FMath::Lerp(Conn.AvgPingMs, PingMs, 0.3f);
            
            OnConnectionQualityChanged.Broadcast(PC, Conn.AvgPingMs);
            
            // 超时检测
            if (PingMs > 1000.f) // > 1s 延迟视为异常
            {
                Conn.ConsecutiveTimeouts++;
                if (Conn.ConsecutiveTimeouts >= MaxConsecutiveTimeouts)
                {
                    UE_LOG(LogGame, Warning, TEXT("Player %s timed out (%.0fms x%d)"),
                        *PC->GetName(), PingMs, MaxConsecutiveTimeouts);
                    OnPlayerTimedOut.Broadcast(PC);
                }
            }
            else
            {
                Conn.ConsecutiveTimeouts = 0;
            }
        }
    }
}
```

**重连机制（建议但不强制实施）**：
- 客户端断开后缓存 PlayerState 10 秒
- 重连时恢复 PlayerState 和 Actor 所有权
- 使用 `AGameModeBase::SwapPlayerControllers()` 无缝切换

### 3.3 预期收益

- 服务器主动检测卡顿/丢包玩家
- UI 可显示延迟指示器
- 为未来大厅/匹配系统预留基础设施

---

## 4. OnlineSubsystem 会话管理

### 4.1 当前状态

`OnlineSubsystem`、`OnlineSubsystemSteam`、`CoreOnline` 在 Build.cs 中声明了依赖，但代码中**从未调用**任何 `IOnlineSubsystem` API。没有 `CreateSession`、`FindSessions`、`JoinSession` 等实现。

### 4.2 改进方案

创建最小的在线会话管理器：

```cpp
// RogueOnlineSubsystem.h

UCLASS(Config=Game)
class URogueOnlineSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
    
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    
    // 会话操作
    void HostSession(int32 MaxPlayers = 4, bool bIsLAN = false);
    void FindSessions();
    void JoinSession(const FOnlineSessionSearchResult& Session);
    void LeaveSession();
    
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnSessionsFound, 
        bool, bSuccess, const TArray<FBlueprintSessionResult>&, Sessions);
    UPROPERTY(BlueprintAssignable)
    FOnSessionsFound OnSessionsFound;
    
private:
    IOnlineSessionPtr SessionInterface;
    
    FOnCreateSessionCompleteDelegate CreateSessionDelegate;
    FOnFindSessionsCompleteDelegate FindSessionsDelegate;
    FOnJoinSessionCompleteDelegate JoinSessionDelegate;
    FOnDestroySessionCompleteDelegate DestroySessionDelegate;
    
    void OnCreateSessionComplete(FName SessionName, bool bSuccess);
    void OnFindSessionsComplete(bool bSuccess);
    void OnJoinSessionComplete(FName SessionName, 
        EOnJoinSessionCompleteResult::Type Result);
};

// 实现
void URogueOnlineSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    
    IOnlineSubsystem* OSS = IOnlineSubsystem::Get();
    if (OSS)
    {
        SessionInterface = OSS->GetSessionInterface();
    }
}

void URogueOnlineSubsystem::HostSession(int32 MaxPlayers, bool bIsLAN)
{
    if (!SessionInterface.IsValid()) return;
    
    FOnlineSessionSettings SessionSettings;
    SessionSettings.bIsLANMatch = bIsLAN;
    SessionSettings.NumPublicConnections = MaxPlayers;
    SessionSettings.bShouldAdvertise = true;
    SessionSettings.bUsesPresence = true;
    SessionSettings.bAllowJoinInProgress = true;
    
    CreateSessionDelegate = FOnCreateSessionCompleteDelegate::CreateUObject(
        this, &ThisClass::OnCreateSessionComplete);
    SessionInterface->AddOnCreateSessionCompleteDelegate_Handle(CreateSessionDelegate);
    
    SessionInterface->CreateSession(0, FName("RogueSession"), SessionSettings);
}

void URogueOnlineSubsystem::OnCreateSessionComplete(FName SessionName, bool bSuccess)
{
    if (bSuccess)
    {
        // 打开监听服务器地图
        GetWorld()->ServerTravel(TEXT("?listen"));
    }
}
```

**在 MainMenu UI 中集成：**
```cpp
// VRogueMainMenu 蓝图中调用
void URogueMainMenu::HostGame()
{
    GetGameInstance()->GetSubsystem<URogueOnlineSubsystem>()->HostSession(4);
}

void URogueMainMenu::FindGames()
{
    auto* OSS = GetGameInstance()->GetSubsystem<URogueOnlineSubsystem>();
    OSS->OnSessionsFound.AddDynamic(this, &URogueMainMenu::OnSessionsFound);
    OSS->FindSessions();
}
```

### 4.3 预期收益

- 从 MainMenu 直接创建/加入多人游戏
- 支持 LAN 和 Steam 两种模式
- 支持游戏中加入（`bAllowJoinInProgress`）

---

## 5. 多人游戏优化细节

### 5.1 网络相关性优化

```cpp
// ARogueProjectile.cpp — 覆盖 IsNetRelevantFor
bool ARogueProjectile::IsNetRelevantFor(const AActor* RealViewer, 
    const AActor* ViewTarget, const FVector& SrcLocation) const
{
    // 只对近处玩家复制（无 ReplicationGraph 时有效）
    const float NetCullDistanceSquared = FMath::Square(10000.f); // 100m
    return FVector::DistSquared(GetActorLocation(), SrcLocation) < NetCullDistanceSquared;
}
```

### 5.2 网络条件 Tick

```cpp
// 投射物移动组件：仅服务器 Tick（客户端由预测驱动）
void URogueProjectileMovementComponent::TickComponent(float DeltaTime, 
    ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    // 服务器做权威模拟，客户端等复制
    if (GetOwner()->HasAuthority())
    {
        Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    }
}
```

### 5.3 对象池网络适配

```cpp
// 当前池在 NM_Standalone 才启用，改为支持 Listen Server
bool URogueActorPoolingSubsystem::IsPoolingEnabled(const UObject* WorldContextObject)
{
    if (!CVarActorPooling.GetValueOnGameThread()) return false;
    
    UWorld* World = WorldContextObject->GetWorld();
    if (!World) return false;
    
    // 改为：客户端不启用，但 Listen Server 和 DS 启用
    return World->GetNetMode() != NM_Client;
}
```

---

## 6. 实施路线图

### 阶段 1：立即实施（1-2 周）

| 优先级 | 改进项 | 工作量 | 收益 |
|-------|-------|--------|------|
| **P0** | 消息系统网络化（`BroadcastTagNet`） | 中 | 客户端 UI/音效事件正确响应 |
| **P0** | `RemoveTagListener` 实现 | 低 | 消除崩溃风险 |
| **P1** | 连接监控 (Ping 检测) | 低 | 可观测性 |
| **P1** | OnlineSubsystem 会话管理 | 中 | 从 UI 创建/加入游戏 |

### 阶段 2：核心优化（2-4 周）

| 优先级 | 改进项 | 工作量 | 收益 |
|-------|-------|--------|------|
| **P1** | ReplicationGraph 实现 | 中 | 网络 CPU 5-10x 减少 |
| **P2** | 对象池网络适配 | 低 | 投射物复用兼容多人 |
| **P2** | 网络相关性优化 (`IsNetRelevantFor`) | 低 | 带宽节省 |

### 阶段 3：扩展（后续迭代）

| 优先级 | 改进项 | 工作量 | 收益 |
|-------|-------|--------|------|
| **P2** | 重连机制（PlayerState 恢复） | 高 | 断线恢复 |
| **P2** | 延迟消息队列（Late Joiner） | 中 | 新加入者状态同步 |
| **P3** | 自定义 Iris 桥接 | 高 | 未来 Iris 全面启用 |

---

**文档版本**: 1.0  
**参考**: PX `UPXReplicationGraph` + `UGMMessageManager` + `UCSNetworkSystem` + `FGCloudOnlineSystem`
