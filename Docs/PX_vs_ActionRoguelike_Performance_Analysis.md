# PX 商业游戏 vs ActionRoguelike 客户端性能优化深度对比分析

## 摘要

本文档基于对 PX 商业游戏项目（及自研 UE5 引擎）客户端性能优化体系和 ActionRoguelike 开源学习项目的深度学习研究，对比两者在对象池、异步计算、Data-Oriented Design（DOD）等关键性能优化方向上的设计理念、实现深度和技术差异，并为 ActionRoguelike 提出参考 PX 项目的具体优化方向。

---

## 第一部分：PX 商业项目客户端性能优化体系

### 1.1 系统架构总览

PX 采用**三层优化体系**：

```
Layer 1: Object Pooling（对象池化）
  - Actor Pool：复用 AActor 实例，减少创建/销毁开销 90%+
  - Widget Pool：复用 UUserWidget 实例，UI 响应延迟降至 <5ms

Layer 2: True Async Creation（真异步创建）
  - 多线程预处理（Worker Thread Pool）
  - 分帧构建（Frame Budget Control）
  - 主线程开销降低 70%+

Layer 3: Intelligent Management（智能管理）
  - LRU 自动清理
  - 内存压力感知
  - 自适应预热
```

### 1.2 Actor 池核心技术

#### 1.2.1 双层池架构

| 特性 | 通用池（Generic） | 专用池（Specialized） |
|-----|------------------|---------------------|
| Actor 类型 | 任意类型 | 单一类型 |
| 队列结构 | `TLockFreePointerListFIFO`（无锁） | 同通用池 |
| 配置 | 全局配置 | 每类独立配置（INI 驱动） |
| 预热 | 手动/自动 | 创建时自动预热 |
| 统计 | 合并统计 | 独立统计 |
| 追踪 | 无 | 追踪所有管理的 Actor |

**关键数据结构：**
- **`FPXObjectFlagsContainer`**：全局 `TMap<uint64, std::atomic<uint32>>` 存储线程安全的标志位（Pooled、AsyncConstruction、PoolAcquiring 等），使用 `FRWLock` 读写锁
- **`FPXCDOSnapshotCache`**：缓存 CDO 属性快照用于快速重置，LRU 淘汰策略，线程安全的读写锁访问
- **`FPXPooledActorReferenceTracker`**：主动通知监听者清理死引用，防止悬空指针

#### 1.2.2 CDO 快照系统（PX 独有）

```cpp
struct FPXCDOSnapshot {
    TMap<FName, TArray<uint8>> PropertyData;  // 属性名 -> 序列化数据
    double LastAccessTime;                     // LRU 淘汰依据
};
```

将 Actor 归还池时，通过 CDO 快照恢复所有属性到默认值，避免逐属性手动重置的遗漏和维护成本。快照在首次使用时创建并缓存，后续复用。

#### 1.2.3 状态重置器（FPXActorStateResetter）

支持三种重置模式：
- **同步重置**：阻塞当前帧，完整重置
- **异步重置**：下一帧 GameThread 执行
- **分帧重置**：分散到多帧执行，避免单帧卡顿

重置步骤：停止移动 → 停止动画蒙太奇 → 清除计时器 → 重置组件状态 → 应用 CDO 快照 → 隐藏并移动到池位置

### 1.3 Widget 池核心技术

PX 的 Widget 池系统与 Actor 池在架构上高度同构，但针对 UMG 特点做了适配：

| 方面 | Actor 池 | Widget 池 |
|-----|---------|----------|
| 注册/注销 | `RegisterComponent` | `AddToViewport`/`RemoveFromParent` |
| 状态重置 | Transform、Physics | 动画、绑定、可见性 |
| 生命周期 | `BeginPlay`/`EndPlay` | `NativeConstruct`/`NativeDestruct` |
| 子对象处理 | 递归重置子组件 | 清理动态子 Widget |

**关键特性：**
- **`IPXPoolableWidget` 接口**：提供 `OnPreAcquireFromPool`、`OnPostReleaseToPool` 等回调
- **`PoolPersistent` 属性**：标记不被重置的关键属性（如 PlayerId）
- **分帧 Slate 构建器**：复杂 Widget 的 Slate 构建分散到多帧（`MaxBuildTimePerFrame = 2.0ms`）
- **CDO 快照恢复**：与 Actor 池相同的机制

### 1.4 真异步（TrueAsync）创建系统（PX 核心创新）

#### 1.4.1 设计理念："准备与应用分离"

```
Worker Thread（并行）              GameThread（串行）
─────────────────────────          ─────────────────────
Phase 1: 内存分配                   Phase 4: Actor 注册
Phase 2: CDO 数据准备               Phase 5: 组件注册
Phase 3: 组件预创建                 Phase 6: 构造脚本执行
                                    Phase 7: BeginPlay
```

#### 1.4.2 线程模型

- **Worker Thread Pool**：内存预分配、CDO 属性深拷贝、组件内存预分配、资源预加载请求、Widget 树快照构建
- **GameThread**：UObject 注册（不可绕过）、组件注册（渲染线程同步）、蓝图构造脚本、BeginPlay
- **通信机制**：MPSC 无锁队列（`TQueue`）、FIFO 无锁队列（`TLockFreePointerListFIFO`）

#### 1.4.3 GC 安全集成

PX 修改了引擎 GC 流程：
```cpp
// GC 钩子 - 引擎级别集成
void FPXAsyncConstructionTracker::OnPreGarbageCollect() {
    // GC 开始前等待所有异步构造完成
    WaitForAllPendingConstructions(5.0f);
}
```

使用 RAII 守卫（`FPXAsyncConstructionGuard`）自动追踪异步构造的生命周期，防止 GC 打断半构造对象。

#### 1.4.4 预分配内存池

```
FPXAsyncObjectAllocator
  └── TMap<FSoftClassPath, FPXClassMemoryPool>
        └── TLockFreePointerListFIFO<FPXPreAllocatedSlot> FreeSlots
```

- 按类分组的无锁内存池
- Worker 线程安全的 O(1) 分配/归还
- 内存所有权转移机制：预分配内存 → 引擎消费 → UObject 系统托管 → GC 释放

#### 1.4.5 帧预算控制

```cpp
// 每帧 GameThread 处理限制
MaxGameThreadProcessPerFrame = 10;  // 数量限制
FrameTimeBudgetMs = 2.0f;           // 时间预算（毫秒）
```

支持动态负载调整：根据当前帧率自动增减处理量和时间预算。

#### 1.4.6 优先级调度

```cpp
enum class EPXSpawnPriority {
    Critical = 0,  // 玩家角色
    High = 1,      // 敌方 NPC
    Normal = 2,    // 普通对象
    Low = 3,       // 装饰物
    Background = 4 // 后台
};
```

每个优先级独立队列，高优先级先处理。

### 1.5 Widget 树快照缓存

PX 独有的 Widget 树快照机制：

```cpp
struct FPXWidgetTreeSnapshot {
    TArray<FPXWidgetNodeSnapshot> Nodes;      // Widget 节点数据
    TArray<FSoftObjectPath> ReferencedAssets; // 资源引用（提前收集加载列表）
    TArray<FPXWidgetBindingSnapshot> Bindings;// 绑定信息（预解析 PropertyPath）
};
```

首次创建后缓存整个 Widget 树结构，后续创建直接从快照恢复，跳过昂贵的 UMG 解析和 Slate 构建阶段。

### 1.6 引擎层修改

PX 对 UE 引擎做了深度定制（6 个核心文件）：

| 文件 | 修改内容 |
|-----|---------|
| `UObjectGlobals.cpp` | 添加 `StaticAllocateObjectAsync()` 异步分配接口 |
| `Obj.cpp` | 支持 PlacementNew 分离式 UObject 构造 |
| `LevelActor.cpp` | `UWorld::SpawnActor()` 添加异步版本 |
| `World.h` | 声明异步 Spawn 接口 |
| `GarbageCollection.cpp` | GC 开始前等待异步构造完成 |
| 新增文件 | `PXAsyncObjectAllocator`、`PXAsyncUObjectConstruction` 等 |

### 1.7 性能基准数据

| 操作 | 无优化 | PX 优化后 | 提升幅度 |
|-----|-------|----------|---------|
| SpawnActor 100个（简单） | 150ms | 8ms | **94.7%** |
| SpawnActor 100个（复杂） | 1500ms | 45ms | **97.0%** |
| CreateWidget 50个 | 250ms | 3ms | **98.8%** |
| 真异步 GT 时间占比 | 100% | 27% | **73% 降低** |
| 内存分配次数 | 10000+ | ~100 | **99%** |
| UI 响应延迟 | 50-100ms | <5ms | **95%** |

---

## 第二部分：ActionRoguelike 项目性能优化体系

### 2.1 项目定位与技术栈

ActionRoguelike（"Project Orion"）是一个 UE5.6 的进阶学习项目，作者 Tom Looman 将其定位为实验性平台，包含 6 个编译期特性开关（`#define`）用于实验不同优化方案。

### 2.2 已实现的性能优化技术清单

#### 2.2.1 对象池系统

| 组件 | 实现 | 状态 |
|-----|------|------|
| `URogueActorPoolingSubsystem` | 基于 `UWorldSubsystem` 的 TArray 池 | 已实现，CVar 控制（默认关） |
| `IRogueActorPoolingInterface` | `PoolBeginPlay()`/`PoolEndPlay()` 接口 | 已实现 |
| `FUserWidgetPool`（引擎内置） | 用于 DamageNumber Widget | 已导入，代码中部分注释 |
| Monster Corpse 池 | 通过 `URogueMonsterCorpseSubsystem` 管理 | 已实现 |

**ActoinRoguelike 池的核心实现：**

```cpp
struct FActorPool {
    TArray<TObjectPtr<AActor>> FreeActors;  // 简单的 TArray 存储
};

// 从池获取：RemoveAtSwap(0, 1, EAllowShrinking::No)
// 归还到池：Disable collision, Hide, Add to FreeActors
// 预热：SpawnActor 后立即 Release 到池
```

**与 PX 的关键差异：**
- 无专用池/通用池分层
- 无 CDO 快照重置（依赖 Actor 自身实现 `PoolEndPlay()` 重置）
- 无无锁队列（使用普通 TArray + 数组操作）
- 无线程安全标志位系统
- 无引用追踪器
- 无 LRU 清理策略
- 无 INI 配置驱动的自动预热
- 返回池时无碰撞/Tick 禁用自动化（需手动在 `PoolEndPlay` 中处理）

#### 2.2.2 聚合 Ticking（Aggregate Ticks）

`URogueTickablesSubsystem` 将多个组件的 Tick 统一到一个 `FTickFunction` 中执行：

```cpp
TArray<FActorComponentTickFunction*> TickableComponents;

void ExecuteTick(...) {
    for (auto* TickFunc : TickableComponents) {
        TickFunc->ExecuteTick(...);  // 单个循环遍历所有组件
    }
}
```

**优势：** 减少引擎 Tick Task Graph 的分发开销。目前仅 `URogueProjectileMovementComponent` 使用。

**PX 对比：** PX 没有显式的 Aggregate Ticking 文档说明，但其 TrueAsync 系统的批量 ParallelFor 处理也达到了类似的批量处理效果。

#### 2.2.3 Significance Manager

实现了完整的 Significance LOD 系统：

- `URogueSignificanceManager`：自定义 significance 管理器
- `URogueSignificanceComponent`：Actor 组件的便利包装
- `IRogueSignificanceInterface`：C++ 接口
- `URogueSignificanceSettings`：配置系统（`UDeveloperSettingsBackedByCVars`）

**AI Character 的具体优化效果：**

| LOD 级别 | 动画更新 | 移动模式 | Mesh LOD | 开销 |
|---------|---------|---------|----------|-----|
| 高（近） | 完整更新 | `MOVE_Walking` | 原始 LOD | 全开销 |
| 低（远） | 减帧/跳过 | `MOVE_NavWalking`（简化寻路） | 强制低 LOD | 大幅降低 |

**实现亮点：**
- 使用 `EPostSignificanceType::Concurrent` 实现线程安全的距离计算
- 用负距离平方 + `WasRecentlyRendered()` 权重优化排序
- 直接管理 `SkeletalMeshComponent` 而非 Actor（缓存友好）
- 与 `IAnimationBudgetAllocator` 集成

#### 2.2.4 Data-Oriented Design（DOD）

**DOD 投射物（`USE_DATA_ORIENTED_PROJECTILES`）：**

完全绕过 Actor 系统，使用纯数据驱动：
```
FProjectileInstance (per-frame update)   → TArray，不复制
    ↓
FProjectileItem (replication data)      → FFastArraySerializer
    ↓  
URogueProjectileData (config)           → UDataAsset
    ↓
URogueProjectilesSubsystem              → UTickableWorldSubsystem 统一 Tick
```

**DOD 金币拾取（`USE_DOD_COIN_PICKUPS`）：**

- 所有金币使用单个 `UInstancedStaticMeshComponent` 渲染（无 Actor 开销）
- 三组平行数组：`CoinPickupLocations[]`、`CoinPickupAmount[]`、`MeshIDs[]`
- `ParallelFor` 多线程距离检测（`USE_MULTITHREADED_COIN_PICKUPS`）
- 使用 `TMpscQueue` 收集并行检测结果
- 最小批次 1000 以平摊 `ParallelFor` 开销
- 先收集后处理的分离策略避免 Cache 污染

#### 2.2.5 延迟任务系统（Deferred Task System）

`URogueDeferredTaskSystem` 基于帧时间预算的延迟执行：

```cpp
constexpr float TargetFPS = 120;
constexpr float MaxFrameBudget = 1.0f / TargetFPS; // ~8.33ms
const double MinBudgetSeconds = 0.001f;             // 1ms 最低消费
```

**与 PX TrueAsync 帧预算控制的对比：** 设计理念相同——基于时间预算的分散执行。但 ActionRoguelike 仅做分帧延迟，PX 还涉及 Worker 线程的预处理。

#### 2.2.6 AI 相关优化

| 技术 | 实现 |
|-----|------|
| Animation Budget Allocator | `USkeletalMeshComponentBudgeted` + 手动启用模块 |
| 骨骼网格优化 | `bUpdateOverlapsOnAnimationFinalize = false` |
| 可见性驱动动画 | `OnlyTickMontagesWhenNotRendered` |
| MID 缓存 | `OverlayHitflashMID` 避免重复创建 |
| AliveMonsters 缓存 | `TArray<TWeakObjectPtr<APawn>>` 快速计数 |
| 材质距离优化 | 非受击时关闭叠加材质绘制距离 |

#### 2.2.7 网络优化

- `FFastArraySerializer` 投射物和金币同步（Delta 序列化，减少带宽）
- `FVector_NetQuantize` 量化位置序列化
- `bReplicateUsingRegisteredSubObjectList`（UE5 子对象复制新特性）
- Iris 网络支持（`SetupIrisSupport`）
- 客户端跳过不需要的 `Tick()`（`IsTickable()` 返回 false on `NM_Client`）

#### 2.2.8 异步加载

- `UAssetManager::LoadPrimaryAsset()` 异步加载怪物 DataAsset
- `LoadAsync()` + 委托回调加载音频、Widget 类
- 遗留问题：部分 `LoadSynchronous()` 调用标注了潜在卡顿风险

#### 2.2.9 微优化

- `RemoveAtSwap` 替代 `RemoveAt`（无序数组 O(1) 删除）
- `EAllowShrinking::No` 防止内存回缩抖动
- `TWeakObjectPtr` 防止悬空引用
- `SCOPED_NAMED_EVENT_FSTRING` / `TRACE_CPUPROFILER_EVENT_SCOPE` 性能追踪
- `TSpscQueue` / `TMpscQueue` 无锁队列

#### 2.2.10 编译期特性开关

```cpp
#define USE_TAGMESSAGING_SYSTEM 0        // 关闭
#define USE_DEFERRED_TASKS 0             // 关闭
#define USE_DATA_ORIENTED_PROJECTILES 0  // 关闭
#define USE_DOD_COIN_PICKUPS 1           // 开启
#define USE_MULTITHREADED_COIN_PICKUPS 1 // 开启
#define USE_MID_HITFLASHOVERLAY 1        // 开启
```

---

## 第三部分：深度对比分析

### 3.1 对象池对比

| 维度 | PX 商业项目 | ActionRoguelike | 差距 |
|-----|-----------|----------------|------|
| **池架构** | 双层（通用+专用），管理器路由 | 单层 TArray | 大 |
| **数据结构** | 无锁 FIFO 队列 | 普通 TArray | 大 |
| **状态重置** | CDO 快照缓存 + 自动化重置器 | 手动实现接口回调 | 大 |
| **线程安全** | 原子标志位 + 读写锁 | 单线程假设 | 大 |
| **引用安全** | 引用追踪器 + 主动通知 | 无机制 | 大 |
| **内存管理** | LRU 清理 + 内存压力响应 + INI 配置 | 无自动清理 | 大 |
| **预热** | 同步/异步 + INI 驱动 + 池未命中自动补充 | PrimeActorPool 手动调用 | 中 |
| **统计监控** | 命中率、并发效率统计 + stat 命令 | TRACE 计数器 | 中 |
| **Widget 池** | 完整实现（与 Actor 池同源） | 部分实现（注释中） | 大 |

### 3.2 异步计算对比

| 维度 | PX TrueAsync 系统 | ActionRoguelike | 差距 |
|-----|------------------|----------------|------|
| **多线程创建** | Worker Thread Pool + GameThread 组装 | 仅 DOD 金币 ParallelFor | 大 |
| **引擎修改** | 6 个核心文件深度修改 | 无引擎修改 | 大 |
| **GC 安全** | GC 钩子 + RAII 守卫 + 等待机制 | 不涉及（无异步 UObject 创建） | 大 |
| **内存预分配** | 按类分组的无锁内存池 | 无 | 大 |
| **帧预算控制** | 数量限制 + 时间预算 + 动态调整 | 固定时间预算（无动态调整） | 中 |
| **优先级调度** | 5 级优先级 + 独立队列 | 无 | 大 |
| **批量处理** | ParallelFor + 共享 CDO 数据 | ParallelFor（金币）+ 单队列处理 | 中 |
| **性能监控** | 各阶段耗时统计 + 并行效率计算 | TRACE 宏 | 中 |

### 3.3 DOD 数据导向设计对比

| 维度 | PX（推断基于架构） | ActionRoguelike | 差距 |
|-----|------------------|----------------|------|
| **投射物 DOD** | 未明确（基于 Actor 池） | 实验性无 Actor 投射物 | ActionRoguelike 独特 |
| **回收物 DOD** | 基于 Actor 池 | ISM + ParallelFor + 平行数组 | ActionRoguelike 独特 |
| **网络同步** | 传统 Actor 复制 + NetGUID 映射 | FFastArraySerializer Delta 复制 | 旗鼓相当 |

**分析：** ActionRoguelike 在 DOD 方面的实验走在前面（无 Actor 投射物、无 Actor 金币），这是 PX 文档中未出现的思路。但 ActionRoguelike 的 DOD 实现是实验性的，未对全部投射物启用。

### 3.4 架构设计哲学对比

| 维度 | PX | ActionRoguelike |
|-----|-----|----------------|
| **设计目标** | 生产级性能，支持大规模战斗 | 教育/实验平台，展示多种技术 |
| **系统完备性** | 完整的企业级系统（文档、测试、CI、CVar） | 功能演示，部分未默认启用 |
| **引擎侵入性** | 深度修改引擎源码 | 零引擎修改，纯项目层实现 |
| **可配置性** | INI 驱动的全配置化 | CVar 控制开关 |
| **技术深度** | 深入到底层内存分配、GC 集成 | 聚焦应用层模式、数据结构 |
| **代码复杂度** | 企业级（数百个类、完整测试体系） | 教育级（清晰的代码注释、单一职责） |

---

## 第四部分：ActionRoguelike 参考 PX 的优化方向

基于以上分析，以下是 ActionRoguelike 可以参考 PX 项目的具体优化建议，按性价比和实现难度排序：

### 4.1 高优先级：低投入高回报

#### 4.1.1 完善 Actor 池系统（参考 PX GenericActorPool）

**当前状态：** 已有基础池框架，但默认关闭（`game.ActorPooling = false`），部分功能不完整。

**优化方向：**
1. **添加 LRU 超时清理**：防止池无限增长。设置 `MaxPooledTimeSeconds`（如 300s），Tick 中检查超时 Actor 并 `Destroy()`
2. **添加最大池容量**：参考 PX 的 `MaxPoolSize` 配置，超出时清理最早入池的 Actor
3. **添加 EAllowShrinking::No**：已在释放时使用，扩展到预热避免首次分配反复 resize
4. **完善 Release 自动化**：Release 时自动执行 `SetActorHiddenInGame(true)`、`SetActorEnableCollision(false)`、`SetActorTickEnabled(false)`，减少调用方忘记处理的风险
5. **启用 CVar 默认值**：稳定后可改为 `game.ActorPooling = true`

```cpp
// 建议扩展 FActorPool 结构
struct FActorPool {
    TArray<TObjectPtr<AActor>> FreeActors;
    double LastAccessTime = 0.0;    // 新增：LRU 追踪
    int32 TotalAcquired = 0;        // 新增：统计
    int32 TotalReleased = 0;        // 新增：统计
};
```

#### 4.1.2 实现 CDO 快照重置（参考 PX CDOSnapshotCache）

**当前状态：** 各 Actor 自行在 `PoolEndPlay()` 中手动重置状态，容易遗漏。

**优化方向：**
- 创建 `FRogueCDOSnapshot` 类，缓存 Actor 的 CDO 属性
- 使用 `TFieldIterator<FProperty>` 遍历所有可重置属性
- 在 Release 时自动应用快照恢复默认值
- 过滤 `Transient`、`SaveGame` 等不应重置的属性
- 这对于 ActionRoguelike 中的 `ARogueProjectile` 尤其有用（需要重置速度、生命周期等）

```cpp
// 伪代码示意
struct FRogueCDOSnapshot {
    TMap<FName, TArray<uint8>> DefaultPropertyData;
    
    void CaptureFromCDO(UClass* Class);
    void ApplyToActor(AActor* Actor, const TSet<FName>& SkipProperties);
};
```

#### 4.1.3 实现 Widget 池化（参考 PX WidgetPool）

**当前状态：** `URogueWorldUserWidget` 中有 `// @todo: WidgetPool.Release(this)` 注释，`FUserWidgetPool` 已导入但大部分被注释。

**优化方向：**
- 为 `URogueDamageNumberWidget` 启用 `FUserWidgetPool`（UE 引擎内置，无需额外实现）
- 为 `URogueWorldUserWidget`（血条等）添加池化支持
- 在 `ARogueHUD` 或 GameMode 中管理 Widget 池生命周期
- 预热常用 Widget（如伤害数字 20 个）

```cpp
// 使用 UE 内置 FUserWidgetPool
FUserWidgetPool DamageNumberPool;
// 初始化时设置池大小
DamageNumberPool.SetWorld(GetWorld());
// 获取
UDamageNumberWidget* Widget = Cast<UDamageNumberWidget>(
    DamageNumberPool.GetOrCreateInstance(DamageNumberClass));
// 归还
DamageNumberPool.Release(Widget);
```

### 4.2 中优先级：显著性能提升

#### 4.2.1 引入无锁队列优化池操作（参考 PX LockFree FIFO）

**当前状态：** 池的 Acquire/Release 使用 TArray 操作（`RemoveAtSwap` 等），在频繁操作时可能有性能开销。

**优化方向：**
- 将 `FActorPool::FreeActors` 从 `TArray` 替换为 `TLockFreePointerListFIFO`（UE 引擎内置）
- 优点：O(1) push/pop，无锁设计，适合高频池操作
- 注意：仅用于同一线程的池操作（ActionRoguelike 目前单线程），为未来多线程预留

#### 4.2.2 实现池化统计监控（参考 PX Stats）

**当前状态：** 仅有 `TRACE_DECLARE_INT_COUNTER` 追踪活跃投射物数量。

**优化方向：**
- 添加池命中率统计
- 添加 `stat ActorPool` 控制台命令
- 在非 Shipping 构建中输出池未命中警告

```cpp
struct FActorPoolStats {
    int32 TotalAcquired = 0;
    int32 TotalReleased = 0;
    int32 PoolHits = 0;    // 从池成功获取
    int32 PoolMisses = 0;  // 池空需要 Spawn
    
    float GetHitRate() const { 
        int32 Total = PoolHits + PoolMisses;
        return Total > 0 ? (float)PoolHits / Total : 0.0f; 
    }
};
```

#### 4.2.3 添加 INI 配置驱动的池预热（参考 PX INI Config）

**当前状态：** `ARogueGameModeBase::RequestPrimedActors()` 硬编码预热配置。

**优化方向：**
- 创建 `URoguePoolingSettings` 继承 `UDeveloperSettings`
- 支持 `TMap<TSubclassOf<AActor>, int32>` 配置默认预热数量
- 在 `DefaultGame.ini` 中可配置

```ini
[/Script/ActionRoguelike.RoguePoolingSettings]
+PrimedActors=(ActorClass=/Game/BP_Projectile_Magic.BP_Projectile_Magic_C,Count=20)
+PrimedActors=(ActorClass=/Game/BP_Projectile_Blackhole.BP_Projectile_Blackhole_C,Count=5)
```

#### 4.2.4 加强 AI 显著性集成（参考 PX 的多层次优化思路）

**当前状态：** `ARogueAICharacter` 有较完善的 Significance 集成，但可以进一步增强。

**优化方向：**
- 添加 PX 类似的**优先级调度**：AI 决策中融入显著性，低 LOD 的 AI 降低行为树评估频率
- 批量 Animation Budget Allocator 配置：按 AI 距离自动调整，而非全局统一
- 考虑 LOD 切换时也修改行为树 Tick 频率

#### 4.2.5 扩展 Aggregate Ticking 到更多组件

**当前状态：** 仅 `URogueProjectileMovementComponent` 使用聚合 Tick。

**优化方向：**
- 将 `URogueActionComponent` 的 Action Effect Tick 也注册到聚合系统
- 将 AI 频繁更新的 `BTService` 相关 Tick 聚合处理
- 为不同组件类型实现**离散排序列表**（如代码注释所提到），提升 Cache 命中率

### 4.3 低优先级：高投入大规模改进

#### 4.3.1 逐步引入分帧创建机制（参考 PX AsyncSpawnStateMachine）

**当前状态：** Deferred Task System 已有帧预算控制雏形，但默认关闭（`USE_DEFERRED_TASKS 0`）。

**优化方向：**
- 首先稳定 Deferred Task System 并启用（`USE_DEFERRED_TASKS 1`）
- 参考 PX 的 `EPXAsyncSpawnState` 状态机，实现 Actor 的**分帧异步 Spawn**
- 将 Spawn 流程拆分为内存分配 → 构造 → 组件注册 → BeginPlay 多个状态
- 每帧处理有限数量的状态转换
- 注意：ActionRoguelike 不应（也不需要）修改引擎源码，可将状态机实现在 Subsystem 层面

#### 4.3.2 考虑 DOD 投射物的生产化（ActionRoguelike 自己的独特优势）

**当前状态：** DOD 投射物实验性实现，默认关闭。

**优化方向：**
- 直接改进和稳定 DOD 投射物系统（这是 ActionRoguelike 相比 PX 的独特技术路线）
- 添加对象池类似的 LRU 清理（清理超时的 `FProjectileInstance`）
- 添加投射物类型的 `TMap<URogueProjectileData*, TArray<FProjectileInstance>>` 分组管理
- 添加闲置投射物槽位复用（类似池的预热，预分配 N 个空槽位）
- **这个方向可能比 PX 的 Actor 池方案在投射物场景更具性能优势**

#### 4.3.3 将 DOD 模式推广到其他高频对象

**当前状态：** 仅金币和实验性投射物使用 DOD。

**优化方向：**
- 评估伤害数字是否可以用类似金币的方式（ISM + 平行数组）实现
- 评估短时效 VFX（如受击闪光）是否可以用纯 Niagara Data Channel 替代 Actor 包装
- **DOD + ISM 模式是 ActionRoguelike 的独特优势**，可以朝着这个方向深入

---

## 第五部分：总结

### 5.1 核心技术对比矩阵

| 技术领域 | PX 商业项目 | ActionRoguelike | 互补价值 |
|---------|-----------|----------------|---------|
| Actor 池 | ★★★★★ 企业级完善 | ★★★☆☆ 基础可用 | PX 的 CDO 快照、LRU 等可直接参考 |
| Widget 池 | ★★★★★ 完整系统 | ★★☆☆☆ 部分实现 | PX 的 Widget 池设计可指导完善 |
| 多线程异步创建 | ★★★★★ 引擎级别 | ★★★☆☆ 应用层实验 | PX 的架构思想可参考，但行为差异大 |
| 帧预算控制 | ★★★★★ 动态调整 | ★★★☆☆ 固定预算 | PX 的动态调整 + 优先级可借鉴 |
| DOD 投射物 | 未明确使用 | ★★★★☆ 实验性 | ActionRoguelike 独特优势 |
| DOD 回收物 | 未明确使用 | ★★★★☆ ISM+ParallelFor | ActionRoguelike 独特优势 |
| Significance | 未明确（推断有） | ★★★★☆ 完善 | ActionRoguelike 已较完善 |
| Aggregate Ticking | 未明确 | ★★★★☆ 已实现 | ActionRoguelike 独特优势 |
| NIagara 池 | 自研池系统 | ★★★★☆ ENCPoolMethod | 成熟方案 |
| 统计监控 | ★★★★★ 完善 | ★★☆☆☆ 基础 | 可大幅提升 |

### 5.2 针对 ActionRoguelike 的核心建议

**立即可以做的（1-2 天工作量）：**
1. 为 Actor 池添加 `MaxPoolSize` 和 LRU 超时清理
2. 为 `URogueDamageNumberWidget` 启用 `FUserWidgetPool`
3. 添加池命中率统计和 debug 输出

**短期可以做的（1-2 周工作量）：**
4. 实现简化版 CDO 快照重置
5. 池数据结构升级为无锁队列
6. 添加 INI 配置驱动的池预热
7. 启用并稳定 Deferred Task System

**中期可以做的（1-2 月工作量）：**
8. 完善 Widget 池化系统（参考 PX WidgetPool 架构）
9. 稳定并生产化 DOD 投射物系统
10. 实现分帧 Actor 创建状态机

**ActionRoguelike 独有的优势方向（持续投入）：**
- **DOD 模式是 ActionRoguelike 的核心差异化竞争力**——继续深化无 Actor 的数据导向设计
- **Aggregate Ticking** —— 扩展覆盖更多组件类型
- **教学性质** —— 可以作为 UE5 性能优化的最佳实践示范项目

### 5.3 关于 PX 项目的启示

PX 项目展示了企业级游戏在以下方面的最佳实践：
1. **系统完备性**：对象池不仅是"省去 Spawn"，而是从 CDO 快照、状态重置、引用追踪、线程安全、GC 集成、内存管理、统计监控的全链路优化
2. **引擎深度定制**：为达到极致性能，PX 选择了修改引擎核心（SpawnActor 流程、GC 流程），但这增加了维护成本
3. **配置驱动**：所有优化策略通过 INI/CVar 可配置、可调优、可降级
4. **文档完善**：每个子系统都有技术原理、数据流转、使用指南、单元测试和性能基准的完整文档

ActionRoguelike 的定位不同，不需要达到 PX 的企业级完备度，但 PX 在对象池化、CDO 快照、Widget 池和帧预算控制方面的设计思想和具体数据值得深入学习和选择性借鉴。

---

**文档版本**: 1.0  
**分析日期**: 2026-06-14  
**分析范围**: PX 商业项目 Docs 全量文档 + PX 自研 UE 引擎修改方案 + ActionRoguelike 全量源码
