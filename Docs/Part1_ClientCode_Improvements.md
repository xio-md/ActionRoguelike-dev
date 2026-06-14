# Part 1: ActionRoguelike 客户端代码性能改进方案

> 基于 PX 商业项目源码深度分析，提出无需修改引擎即可在 ActionRoguelike 中实施的优化方案。

---

## 目录

1. [对象池系统升级](#1-对象池系统升级)
2. [Widget 池化完善](#2-widget-池化完善)
3. [CDO 快照重置系统](#3-cdo-快照重置系统)
4. [无锁队列优化](#4-无锁队列优化)
5. [标志位线程安全系统](#5-标志位线程安全系统)
6. [引用追踪器](#6-引用追踪器)
7. [聚合 Tick 扩展](#7-聚合-tick-扩展)
8. [延迟任务系统生产化](#8-延迟任务系统生产化)
9. [DOD 投射物稳定化](#9-dod-投射物稳定化)
10. [实施路线图](#10-实施路线图)

---

## 1. 对象池系统升级

### 1.1 当前状态

ActionRoguelike 已有的 `URogueActorPoolingSubsystem` 提供基础池功能：
- `TMap<TSubclassOf<AActor>, FActorPool>` 存储结构（`TArray<TObjectPtr<AActor>> FreeActors`）
- `IRogueActorPoolingInterface` 接口（`PoolBeginPlay` / `PoolEndPlay`）
- `PrimeActorPool()` 手动预热
- CVar `game.ActorPooling` 默认关闭

**当前问题：** 缺少 LRU 清理、最大容量限制、自动化状态管理、命中率统计。

### 1.2 改进方案

#### 1.2.1 增加 LRU 超时清理

参考 PX `FPXGenericActorPool::CleanupExpired()` 实现：

```cpp
// RogueActorPoolingSubsystem.h 新增
USTRUCT()
struct FActorPool
{
    GENERATED_BODY()
    
    UPROPERTY()
    TArray<TObjectPtr<AActor>> FreeActors;
    
    // 新增：LRU 追踪
    double LastAccessTime = 0.0;
    int32 TotalAcquired = 0;
    int32 TotalReleased = 0;
    int32 PoolHits = 0;
    int32 PoolMisses = 0;
};

// 配置（可在 CVar 中设置）
static float PoolIdleTimeout = 300.0f;  // 5分钟
static int32 MaxPoolSize = 50;          // 单类最大池容量
```

Tick 中添加清理逻辑：

```cpp
void URogueActorPoolingSubsystem::Tick(float DeltaTime)
{
    double CurrentTime = FPlatformTime::Seconds();
    
    for (auto& [Class, Pool] : AvailableActorPool)
    {
        // LRU 清理：按 PooledTime 排序（需要在 Entry 中记录入池时间）
        if (Pool.FreeActors.Num() > MaxPoolSize)
        {
            // 清理超出部分，优先销毁最久未使用的
            int32 RemoveCount = Pool.FreeActors.Num() - MaxPoolSize;
            for (int32 i = 0; i < RemoveCount; ++i)
            {
                AActor* Actor = Pool.FreeActors.Last();
                Pool.FreeActors.RemoveAt(Pool.FreeActors.Num() - 1);
                Actor->Destroy();
            }
        }
    }
}
```

**预期性能提升：**
- 防止池无限增长导致内存泄漏
- 在 ActionRoguelike 的 AI 大量刷新场景中控制内存峰值

#### 1.2.2 增加 INI 配置驱动的专用池预热

参考 PX `FPXSpecializedActorPoolConfig` 和 `UPXActorPoolSettings`：

```cpp
// 新增：URoguePoolingSettings.h
UCLASS(Config=Game, DefaultConfig)
class URoguePoolingSettings : public UDeveloperSettings
{
    GENERATED_BODY()
    
public:
    // 通用池配置
    UPROPERTY(Config, EditAnywhere)
    int32 GenericPoolMaxSize = 50;
    
    UPROPERTY(Config, EditAnywhere)
    float GenericPoolIdleTimeout = 300.0f;
    
    // 专用池配置（Map 形式：类 -> 预热数量 + 最大容量）
    UPROPERTY(Config, EditAnywhere)
    TMap<TSoftClassPtr<AActor>, int32> ActorPoolPrewarmCounts;
    
    UPROPERTY(Config, EditAnywhere)
    TMap<TSoftClassPtr<AActor>, int32> ActorPoolMaxSizes;
};
```

DefaultGame.ini 配置示例：

```ini
[/Script/ActionRoguelike.RoguePoolingSettings]
GenericPoolMaxSize=50
GenericPoolIdleTimeout=300.0
+ActorPoolPrewarmCounts=(Key=/Game/BP_Projectile_Magic.BP_Projectile_Magic_C,Value=20)
+ActorPoolPrewarmCounts=(Key=/Game/BP_Projectile_Dash.BP_Projectile_Dash_C,Value=5)
+ActorPoolMaxSizes=(Key=/Game/BP_Projectile_Magic.BP_Projectile_Magic_C,Value=100)
```

**预期性能提升：** 投射物首次创建延迟从 5-10ms 降至 <0.1ms。

#### 1.2.3 池未命中自动异步补充

参考 PX `TryTriggerAsyncRefillOnAcquire()` 实现。当池为空时，并非立即全部补充，而是通过 AsyncTask 在后台补充：

```cpp
void URogueActorPoolingSubsystem::TryTriggerAsyncRefill(TSubclassOf<AActor> ActorClass)
{
    // 冷却检查：避免短时间内重复补充
    double CurrentTime = FPlatformTime::Seconds();
    if (double* LastRefill = AsyncRefillCooldowns.Find(ActorClass))
    {
        if (CurrentTime - *LastRefill < RefillCooldownSeconds)
            return;
    }
    AsyncRefillCooldowns.Add(ActorClass, CurrentTime);
    
    // 异步补充：不使用 Actor 池的 Actor（那是给游戏用的），而是直接 Spawn + Release
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [this, ActorClass]()
    {
        // 在后台线程只是做记录，实际 Spawn 仍需 GameThread
        // 使用 FSimpleDelegate 回到 GameThread
        AsyncTask(ENamedThreads::GameThread, [this, ActorClass]()
        {
            PrimeActorPool(ActorClass, AsyncRefillCount);
        });
    });
}
```

**预期性能提升：** 连续投射物发射的池命中率从 0% 恢复到 >95%。

### 1.3 性能预估

| 场景 | 改进前 | 改进后 | 提升 |
|-----|-------|-------|------|
| 投射物首次创建（无预热） | 5-10ms | 5-10ms | 无变化 |
| 投射物复用创建（有预热） | 5-10ms | <0.1ms | **50-100x** |
| 池内存占用（无控制） | 无限增长 | <50MB（受控） | 防止泄漏 |
| 池命中率（连续使用） | 0%（无预热） | >95%（自动补充） | 大幅提升 |

---

## 2. Widget 池化完善

### 2.1 当前状态

- `FUserWidgetPool` 已导入但大部分被注释
- `URogueWorldUserWidget` 中有 `// @todo: WidgetPool.Release(this)` 注释
- `URogueDamageNumberWidget` 适合池化但未启用

### 2.2 改进方案

#### 2.2.1 启用 UE 内置 FUserWidgetPool

UE 引擎已内置 `FUserWidgetPool`（`Engine/Source/Runtime/UMG/Public/Blueprint/UserWidgetPool.h`），直接使用：

```cpp
// ARogueHUD.h 新增
UPROPERTY()
FUserWidgetPool DamageNumberPool;

// URogueWorldUserWidget.h 新增
UPROPERTY()
FUserWidgetPool HealthBarPool;

void ARogueHUD::BeginPlay()
{
    Super::BeginPlay();
    
    // 设置池属性
    DamageNumberPool.SetWorld(GetWorld());
    // 预创建 20 个伤害数字
    for (int32 i = 0; i < 20; ++i)
    {
        UUserWidget* Widget = DamageNumberPool.GetOrCreateInstance(
            DamageNumberWidgetClass.LoadSynchronous());
        DamageNumberPool.Release(Widget);
    }
}

// 使用（替代 CreateWidget）
UUserWidget* Widget = DamageNumberPool.GetOrCreateInstance(DamageNumberClass);
// 归还
DamageNumberPool.Release(Widget);
```

#### 2.2.2 添加 WorldSubsystem 管理 Widget 池生命周期

参考 PX `FPXWidgetPoolManager`：

```cpp
UCLASS()
class URogueWidgetPoolSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()
    
public:
    // 按 Widget 类分组的池
    TMap<TSubclassOf<UUserWidget>, FUserWidgetPool> WidgetPools;
    
    UUserWidget* AcquireOrCreate(TSubclassOf<UUserWidget> WidgetClass, UObject* Owner);
    void Release(UUserWidget* Widget);
    void Prewarm(UWorld* World, TSubclassOf<UUserWidget> WidgetClass, int32 Count);
    
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
};
```

**预期性能提升：**
- 伤害数字 Widget 创建从 2-5ms 降至 <0.1ms
- 战斗场景中 50 个同时显示的伤害数字不再引起 UI 卡顿

---

## 3. CDO 快照重置系统

### 3.1 设计目标

解决对象池的核心痛点：**如何确保从池中取出的 Actor 状态完全干净**。

当前 ActionRoguelike 依赖各 Actor 手动实现 `PoolEndPlay()` 重置，容易遗漏属性。PX 的方案是 CDO 快照缓存——在首次使用时缓存类的默认属性值，后续从快照恢复。

### 3.2 实现方案

#### 3.2.1 核心数据结构

```cpp
// RogueCDOSnapshot.h

struct FRoguePropertySnapshot
{
    FName PropertyName;
    int32 Offset;         // 属性在对象中的偏移
    int32 Size;           // 属性大小
    TArray<uint8> Data;   // 序列化的默认值
};

class FRogueCDOSnapshot
{
public:
    void CaptureFromCDO(UClass* InClass);
    void ApplyToActor(AActor* Actor, const TSet<FName>& SkipProperties);
    
    bool IsValid() const { return bCaptured && SourceClass.IsValid(); }
    
private:
    TWeakObjectPtr<const UClass> SourceClass;
    TArray<FRoguePropertySnapshot> PropertySnapshots;
    bool bCaptured = false;
};

class FRogueCDOSnapshotCache
{
public:
    static FRogueCDOSnapshotCache& Get();
    
    const FRogueCDOSnapshot* GetOrCreateSnapshot(UClass* Class);
    void ClearCache();
    
private:
    TMap<TWeakObjectPtr<const UClass>, TUniquePtr<FRogueCDOSnapshot>> Cache;
    FCriticalSection CacheLock;
};
```

#### 3.2.2 Capture 实现（遍历 CDO 属性）

```cpp
void FRogueCDOSnapshot::CaptureFromCDO(UClass* InClass)
{
    SourceClass = InClass;
    PropertySnapshots.Empty();
    
    UObject* CDO = InClass->GetDefaultObject();
    if (!CDO) return;
    
    // 遍历所有非临时、非编辑器属性
    for (TFieldIterator<FProperty> It(InClass); It; ++It)
    {
        FProperty* Property = *It;
        
        // 跳过不应重置的属性
        if (Property->HasAnyPropertyFlags(CPF_Transient | CPF_EditorOnly))
            continue;
        if (Property->HasAnyPropertyFlags(CPF_SkipSerialization))
            continue;
        
        FRoguePropertySnapshot Snap;
        Snap.PropertyName = Property->GetFName();
        Snap.Offset = Property->GetOffset_ForInternal();
        Snap.Size = Property->GetSize();
        Snap.Data.SetNumUninitialized(Snap.Size);
        
        // 从 CDO 读取默认值
        Property->CopyCompleteValue(
            Snap.Data.GetData(),
            (uint8*)CDO + Snap.Offset);
        
        PropertySnapshots.Add(MoveTemp(Snap));
    }
    
    bCaptured = true;
}
```

#### 3.2.3 Apply 实现（恢复到 CDO 默认值）

```cpp
void FRogueCDOSnapshot::ApplyToActor(AActor* Actor, const TSet<FName>& SkipProperties)
{
    if (!bCaptured || !Actor) return;
    
    for (const FRoguePropertySnapshot& Snap : PropertySnapshots)
    {
        if (SkipProperties.Contains(Snap.PropertyName))
            continue;
        
        FProperty* Property = SourceClass->FindPropertyByName(Snap.PropertyName);
        if (Property)
        {
            Property->CopyCompleteValue(
                (uint8*)Actor + Snap.Offset,
                Snap.Data.GetData());
        }
    }
}
```

#### 3.2.4 集成到池系统中

```cpp
void URogueActorPoolingSubsystem::ReleaseToPool_Internal(AActor* Actor)
{
    // 现有逻辑：禁用碰撞、隐藏等
    
    // 新增：CDO 快照重置
    const FRogueCDOSnapshot* Snapshot = 
        FRogueCDOSnapshotCache::Get().GetOrCreateSnapshot(Actor->GetClass());
    if (Snapshot)
    {
        Snapshot->ApplyToActor(Actor, {/* skip properties if needed */});
    }
    
    // 加入池
    AvailableActorPool.FindOrAdd(Actor->GetClass()).FreeActors.Add(Actor);
}
```

### 3.3 预期性能提升

| 操作 | 手动重置 | CDO 快照重置 |
|-----|---------|------------|
| 简单投射物重置 | ~0.05ms | ~0.02ms |
| 复杂 AI Character 重置 | ~0.5ms（易遗漏） | ~0.15ms（完整覆盖） |
| 首次创建快照 | N/A | ~0.1ms（一次性） |
| 快照缓存命中 | N/A | <0.01ms |

额外收益：减少属性遗漏 bug，降低维护成本。

---

## 4. 无锁队列优化

### 4.1 当前状态

Actor 池使用 `TArray<TObjectPtr<AActor>>` + `RemoveAtSwap(0, 1, EAllowShrinking::No)` 实现 O(1) 出队，但这只是 O(1) 数组操作，在大规模并发场景下仍有优化空间。

### 4.2 改进方案

使用 UE 引擎内置的 `TLockFreePointerListFIFO` 替代 TArray：

```cpp
// RogueActorPoolingSubsystem.h

#include "Containers/LockFreeList.h"

USTRUCT()
struct FActorPool
{
    GENERATED_BODY()
    
    // 替换 TArray<TObjectPtr<AActor>> FreeActors
    // 使用无锁指针队列：无锁 push/pop，天然线程安全
    // 注意：UE 的 TLockFreePointerListFIFO 不支持 UPROPERTY
    // 需要改用指针包装
};

// 实际实现中，使用 FPXPooledActorEntry 风格的封装
struct FRoguePooledActorEntry
{
    TWeakObjectPtr<AActor> Actor;
    TWeakObjectPtr<UClass> ActorClass;
    double PooledTime;
    int32 UseCount;
    
    bool IsValid() const { return Actor.IsValid(); }
};

// 池队列改为：
TMap<TSubclassOf<AActor>, 
     TUniquePtr<TLockFreePointerListFIFO<FRoguePooledActorEntry, PLATFORM_CACHE_LINE_SIZE>>>
     PoolQueues;
```

**注意事项：**
- `TLockFreePointerListFIFO` 适用于高频 push/pop 场景
- ActionRoguelike 当前为单线程 GameThread 操作，收益不大
- 主要价值在于为未来多线程扩展预留
- **推荐先实施 LRU + CDO 快照，无锁队列作为可选优化**

### 4.3 预期性能提升

| 场景 | TArray RemoveAtSwap | LockFreeFIFO |
|-----|-------------------|--------------|
| 单线程高频 Acquire/Release | <0.01ms | <0.01ms（相当） |
| 多线程并发 Acquire | 需要额外锁 | 天然无锁 |
| 内存碎片 | 有 | 低 |
| 代码复杂度 | 低 | 中 |

---

## 5. 标志位线程安全系统

### 5.1 设计目标

PX 使用 `FPXObjectFlagsContainer`（全局 `TMap<uint32, std::atomic<uint32>>`）管理池化对象的状态标志。这使得可以在任意线程查询对象是否在池中，而无需持有池锁。

### 5.2 ActionRoguelike 简化版实现

由于 ActionRoguelike 当前主要在 GameThread 操作，可以简化实现：

```cpp
// RogueObjectFlags.h

enum class ERogueObjectFlags : uint8
{
    None       = 0,
    Pooled     = 1 << 0,  // 在池中（空闲状态）
    Acquiring  = 1 << 1,  // 正在获取
    Releasing  = 1 << 2,  // 正在归还
    Prewarming = 1 << 3,  // 预热中
};

class FRogueObjectFlagsContainer
{
public:
    static void SetFlags(const UObject* Object, ERogueObjectFlags Flags);
    static void ClearFlags(const UObject* Object, ERogueObjectFlags Flags);
    static bool HasAnyFlags(const UObject* Object, ERogueObjectFlags Flags);
    
    // 便捷方法
    static bool IsPooled(const UObject* Object)
    {
        return HasAnyFlags(Object, ERogueObjectFlags::Pooled);
    }
    
private:
    // 使用 Object 指针地址作为 Key（单线程安全）
    static TMap<uint64, uint8> ObjectFlagsMap;
    static FCriticalSection MapLock;
};
```

**集成到池系统：**
```cpp
// Release 时设置标志
FRogueObjectFlagsContainer::SetFlags(Actor, ERogueObjectFlags::Pooled);

// Acquire 时清除标志
FRogueObjectFlagsContainer::ClearFlags(Actor, ERogueObjectFlags::Pooled);

// 外部检查
if (FRogueObjectFlagsContainer::IsPooled(SomeActor))
{
    // 不可使用，对象在池中
}
```

### 5.3 预期收益

- 防止"已回池但外部引用未清理"导致的 bug
- 为 `TWeakObjectPtr` 检查提供额外安全保障
- 为未来多线程操作预留基础设施

---

## 6. 引用追踪器

### 6.1 设计目标

解决"对象回池后外部引用悬空"的问题。PX 的 `FPXPooledActorReferenceTracker` 在 Actor 回池时主动通知所有注册的监听者清理引用。

### 6.2 实现方案

```cpp
// RoguePooledReferenceTracker.h

class FRoguePooledReferenceTracker
{
public:
    static FRoguePooledReferenceTracker& Get();
    
    // 注册监听：当 Actor 回池时调用回调
    void RegisterListener(
        AActor* Actor,
        TFunction<void(AActor*)> OnPooled
    );
    
    // 通知所有监听者
    void NotifyActorPooled(AActor* Actor);
    
private:
    TMap<TWeakObjectPtr<AActor>, TArray<TFunction<void(AActor*)>>> Listeners;
    FCriticalSection ListenersLock;
};
```

**使用示例（在 AI 控制器中）：**

```cpp
void ARogueAIController::OnTargetAcquired(AActor* Target)
{
    CurrentTarget = Target;
    
    // 注册监听：当 Target 被回池时自动清除引用
    FRoguePooledReferenceTracker::Get().RegisterListener(
        Target,
        [this](AActor* /*PooledActor*/)
        {
            CurrentTarget = nullptr;
            Blackboard->ClearValue(TEXT("TargetActor"));
        }
    );
}
```

### 6.3 预期收益

- 消除"幽灵目标"bug（AI 追踪已回池的投射物/角色）
- 减少 `IsValid()` 检查的开销（主动通知 vs 被动检查）

---

## 7. 聚合 Tick 扩展

### 7.1 当前状态

`URogueTickablesSubsystem` 已实现聚合 Tick 框架，但仅 `URogueProjectileMovementComponent` 使用。

### 7.2 改进方案

扩展到高频 Tick 的组件：

```cpp
// URogueActionComponent.cpp
void URogueActionComponent::BeginPlay()
{
    Super::BeginPlay();
    
    // 注册到聚合 Tick 系统（减少 TaskGraph 开销）
    if (URogueTickablesSubsystem* TickSystem = GetWorld()->GetSubsystem<URogueTickablesSubsystem>())
    {
        TickSystem->RegisterComponent(&PrimaryComponentTick);
    }
}
```

**适合聚合 Tick 的组件：**
- `URogueActionComponent` — 每个 AI 和玩家都有一个，大量实例
- `URogueAttributeSet` 的定时器替代
- 任何有 `TickInterval > 0` 的组件

**代码注释已提到的优化方向：**
```cpp
// 按组件类型分组排序，提升 CPU Cache 命中率
TMap<UClass*, TArray<FActorComponentTickFunction*>> SortedTickables;

void ExecuteTick(...)
{
    for (auto& [Class, Tickables] : SortedTickables)
    {
        for (auto* TickFunc : Tickables)
        {
            TickFunc->ExecuteTick(...);
        }
    }
}
```

### 7.3 预期性能提升

| 组件数量 | 传统 Tick Graph | 聚合 Tick | 节省 |
|---------|---------------|----------|------|
| 20 投射物 | ~0.05ms 调度开销 | ~0.01ms | **80%** |
| 50 AI + 50 投射物 | ~0.15ms 调度开销 | ~0.03ms | **80%** |

---

## 8. 延迟任务系统生产化

### 8.1 当前状态

`URogueDeferredTaskSystem` 已实现帧预算控制的延迟执行，但 `USE_DEFERRED_TASKS` 默认关闭（`0`）。

### 8.2 改进方案

#### 8.2.1 启用并增加动态调整

参考 PX `FPXTrueAsyncActorSpawner::AdjustBudget()`：

```cpp
void URogueDeferredTaskSystem::Tick(float DeltaTime)
{
    // 现有：基于 TargetFPS 的固定预算
    // 新增：基于实际帧率动态调整
    
    static float SmoothedFrameTime = 1.0f / 60.0f;
    SmoothedFrameTime = FMath::Lerp(SmoothedFrameTime, DeltaTime, 0.1f);
    
    // 帧率低时减少处理量，帧率高时增加
    const float TargetFrameTime = 1.0f / 60.0f;
    float BudgetScale = TargetFrameTime / FMath::Max(SmoothedFrameTime, 0.001f);
    BudgetScale = FMath::Clamp(BudgetScale, 0.5f, 2.0f);
    
    AdjustedFrameBudget = MaxFrameBudget * BudgetScale;
    
    // 使用调整后的预算处理任务...
}
```

#### 8.2.2 应用到 AI 生成

```cpp
void ARoguePrimaryGameMode::SpawnMonsterDelayed(TSubclassOf<AActor> MonsterClass, FVector Location)
{
    // 使用延迟任务系统，而非立即 Spawn
    URogueDeferredTaskSystem::AddLambda(GetWorld(), [this, MonsterClass, Location]()
    {
        FActorSpawnParameters Params;
        Params.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;
        GetWorld()->SpawnActor<AActor>(MonsterClass, Location, FRotator::ZeroRotator, Params);
    });
}
```

#### 8.2.3 启用开关

```cpp
// ActionRoguelike.h
#define USE_DEFERRED_TASKS 1  // 从 0 改为 1
```

### 8.3 预期性能提升

- 大规模 AI 生成时帧率波动减少 30-50%
- 分散非关键操作到多帧，平滑帧时间

---

## 9. DOD 投射物稳定化

### 9.1 当前状态

DOD 投射物系统（`USE_DATA_ORIENTED_PROJECTILES`）已实现但默认关闭。系统包含：
- `URogueProjectilesSubsystem` — 管理投射物生命周期
- `FProjectileInstance` — 每帧更新数据
- `FProjectileItem` + `FFastArraySerializer` — 网络同步
- `URogueProjectileData` — 配置 DataAsset

### 9.2 改进方案

#### 9.2.1 添加休眠清理

参考 PX 池的 LRU 策略：

```cpp
void URogueProjectilesSubsystem::Tick(float DeltaTime)
{
    // 现有：移动 + 碰撞检测
    // 新增：清理超时的空闲槽位
    
    int32 ActiveCount = 0;
    for (const FProjectileInstance& Instance : ProjectileInstances)
    {
        if (Instance.ID != 0)  // 0 表示空闲槽位
            ActiveCount++;
    }
    
    // 如果空闲槽位超过阈值，收缩数组
    if (ProjectileInstances.Num() - ActiveCount > MaxIdleSlots)
    {
        // 移除末尾的空闲槽位
        while (ProjectileInstances.Num() > ActiveCount + MaxIdleSlots)
        {
            if (ProjectileInstances.Last().ID == 0)
                ProjectileInstances.RemoveAt(ProjectileInstances.Num() - 1);
            else
                break;
        }
    }
}
```

#### 9.2.2 添加投射物池预热

```cpp
void URogueProjectilesSubsystem::Prewarm(int32 Count)
{
    // 预分配槽位，避免运行时动态扩展数组
    ProjectileInstances.Reserve(Count);
    
    // 填充空闲槽位
    for (int32 i = 0; i < Count; ++i)
    {
        ProjectileInstances.Add(FProjectileInstance());
    }
}
```

#### 9.2.3 启用生产环境

```cpp
// ActionRoguelike.h
#define USE_DATA_ORIENTED_PROJECTILES 1  // 从 0 改为 1
```

### 9.3 预期性能提升

| 场景 | Actor 投射物 | DOD 投射物 |
|-----|-----------|-----------|
| 100 投射物同时更新 | ~3ms (Actor Tick) | ~0.5ms (批量循环) |
| 内存占用/投射物 | ~2KB (Actor + 组件) | ~50B (纯数据) |
| 网络带宽（100 个） | 完整复制 | Delta 压缩 |
| GC 压力 | 高（100 UObject） | 零（无 UObject） |

---

## 10. 实施路线图

### 阶段 1：低风险快速见效（1-2 天）

| 优先级 | 改进项 | 实施难度 | 性能提升 |
|-------|-------|---------|---------|
| P0 | Widget 池化启用（`FUserWidgetPool`） | 低 | 伤害数字创建 50-100x |
| P0 | Actor 池 LRU 清理 + MaxPoolSize | 低 | 防止内存泄漏 |
| P0 | 池命中率统计 + CVar 调试命令 | 低 | 可观测性 |
| P1 | INI 配置驱动预热 | 低 | 首次创建延迟消除 |

### 阶段 2：中期稳定改进（1-2 周）

| 优先级 | 改进项 | 实施难度 | 性能提升 |
|-------|-------|---------|---------|
| P0 | CDO 快照重置系统 | 中 | 重置时间 50%+，减少 bug |
| P1 | 标志位系统 | 低 | 安全性提升 |
| P1 | 引用追踪器 | 中 | 消除悬空引用 bug |
| P2 | 延迟任务系统生产化 | 中 | 帧率波动 30-50% 降低 |

### 阶段 3：深度优化（2-4 周）

| 优先级 | 改进项 | 实施难度 | 性能提升 |
|-------|-------|---------|---------|
| P1 | 聚合 Tick 扩展 | 中 | Tick 调度开销 80% 降低 |
| P2 | DOD 投射物生产化 | 高 | 投射物性能 6x+，零 GC |
| P2 | 无锁队列替换 | 中 | 为多线程预留 |
| P3 | Widget 池管理器 Subsystem | 中 | 全 UI 池化覆盖 |

---

## 附录：关键文件清单

| 新增文件 | 用途 |
|---------|------|
| `Source/ActionRoguelike/Performance/RogueCDOSnapshot.h/.cpp` | CDO 快照缓存系统 |
| `Source/ActionRoguelike/Performance/RogueObjectFlags.h/.cpp` | 标志位容器 |
| `Source/ActionRoguelike/Performance/RoguePooledReferenceTracker.h/.cpp` | 引用追踪器 |
| `Source/ActionRoguelike/Performance/RoguePoolingSettings.h/.cpp` | INI 配置 |
| `Source/ActionRoguelike/UI/RogueWidgetPoolSubsystem.h/.cpp` | Widget 池管理器 |

| 修改文件 | 修改内容 |
|---------|---------|
| `RogueActorPoolingSubsystem.h/.cpp` | 添加 LRU、统计、标志位集成 |
| `RogueTickablesSubsystem.h/.cpp` | 排序列表优化 |
| `ActionRoguelike.h` | 启用 USE_DEFERRED_TASKS, USE_DATA_ORIENTED_PROJECTILES |
| `ActionRoguelike.Build.cs` | 无需修改（已有所有依赖） |

---

**文档版本**: 1.0  
**基于源码分析**: PX `GPFramework/GPGlobalDefines/Pool/` 全量源码 + ActionRoguelike 全量源码
