# GAS面试准备

Unreal Engine Gameplay Ability System 面试题整理，从基础到高级。

---

## 一、基础概念

### 1. GAS的核心组件有哪些？各自职责是什么？

| 组件 | 职责 |
|------|------|
| **AbilitySystemComponent (ASC)** | 核心管理器，挂载在Actor上，管理所有GA、GE、Attribute |
| **GameplayAbility (GA)** | 技能逻辑载体，定义技能的执行流程 |
| **GameplayEffect (GE)** | 数值修改器，修改Attribute、施加Tag、触发Cue |
| **AttributeSet** | 属性集合，存储血量、攻击力等数值 |
| **GameplayTag** | 轻量级标签，用于状态标记、技能分类、条件判断 |
| **GameplayCue** | 表现层，处理特效、音效等客户端表现 |

---

### 2. GameplayEffect的三种Duration类型区别？

| 类型 | 特点 | 典型用途 |
|------|------|----------|
| **Instant** | 立即执行，不持续 | 直接伤害、治疗 |
| **Duration** | 持续一段时间后自动移除 | Buff、Debuff |
| **Infinite** | 永久存在，需手动移除 | 被动技能、装备属性 |

---

### 3. GameplayTag的设计思想和作用？

**设计思想**：数据驱动的层级标签系统，替代硬编码的枚举和字符串。

**核心作用**：
- 技能分类：`Ability.Skill.Fire`
- 状态标记：`State.Debuff.Stun`
- 条件判断：技能释放条件、Effect应用条件
- 事件触发：`GameplayEvent.Damage`

**优势**：
- 层级结构支持模糊匹配
- 编辑器友好，策划可配置
- 无需改代码即可扩展

---

### 4. Attribute和MetaAttribute的区别？

| 类型 | 存储 | 同步 | 用途 |
|------|------|------|------|
| **Attribute** | 持久存储 | 网络同步 | Health、Mana、Attack |
| **MetaAttribute** | 临时存在 | 不同步 | IncomingDamage、IncomingHeal |

**MetaAttribute典型用途**：
```cpp
// 伤害计算流程
1. ExecutionCalculation计算原始伤害 → 写入Meta.IncomingDamage
2. PostGameplayEffectExecute读取IncomingDamage → 应用到Health
3. IncomingDamage清零（临时值，不保留）
```

---

## 二、中级实践

### 5. 完整的伤害流程是怎样的？

```
GA触发攻击
    ↓
创建GE Spec，填充Context（来源、命中信息）
    ↓
应用GE到目标ASC
    ↓
ExecutionCalculation计算伤害（读取攻防、暴击等）
    ↓
修改Attribute（写入MetaAttribute或直接扣血）
    ↓
PostGameplayEffectExecute处理最终逻辑（死亡判定等）
    ↓
触发GameplayCue播放表现（伤害数字、特效）
```

---

### 6. AbilityTask是什么？为什么需要它？

**定义**：GA内的异步任务单元，用于处理需要等待的操作。

**为什么需要**：
- GA的`ActivateAbility`是同步函数
- 很多操作需要等待：播放动画、等待输入、延时

**常用Task**：
| Task | 用途 |
|------|------|
| `PlayMontageAndWait` | 播放动画并等待 |
| `WaitGameplayEvent` | 等待事件触发 |
| `WaitInputPress` | 等待玩家输入 |
| `WaitDelay` | 延时 |
| `WaitTargetData` | 等待目标选择 |

**示例**：
```cpp
void UMyAbility::ActivateAbility(...)
{
    UAbilityTask_PlayMontageAndWait* Task = 
        UAbilityTask_PlayMontageAndWait::CreatePlayMontageAndWaitProxy(
            this, NAME_None, AttackMontage);
    
    Task->OnCompleted.AddDynamic(this, &UMyAbility::OnMontageCompleted);
    Task->OnCancelled.AddDynamic(this, &UMyAbility::OnMontageCancelled);
    Task->ReadyForActivation();
}
```

---

### 7. GE的Stacking（堆叠）策略有哪些？

| 策略 | 说明 |
|------|------|
| **None** | 不堆叠，每次独立应用 |
| **AggregateBySource** | 按来源聚合，同一来源只保留一个 |
| **AggregateByTarget** | 按目标聚合，目标身上只保留一个 |

**堆叠配置项**：
- `StackLimitCount`：最大堆叠层数
- `StackDurationRefreshPolicy`：刷新时是否重置持续时间
- `StackPeriodResetPolicy`：刷新时是否重置周期
- `StackExpirationPolicy`：到期时移除多少层

---

### 8. 技能的冷却和消耗怎么实现？

**冷却（Cooldown）**：
```cpp
// GA中配置CooldownGameplayEffectClass
// GE配置：
// - Duration类型
// - GrantedTags添加冷却Tag: Ability.Cooldown.MySkill
// - 技能激活条件检查该Tag
```

**消耗（Cost）**：
```cpp
// GA中配置CostGameplayEffectClass
// GE配置：
// - Instant类型
// - Modifier减少对应Attribute（如Mana）
// - CheckCost()自动检查资源是否足够
```

---

### 9. Prediction Key的作用？

**问题**：网络延迟导致技能表现滞后

**解决**：客户端预测 + 服务器校验

**流程**：
```
客户端：
1. 生成PredictionKey
2. 立即执行技能（预测）
3. 发送请求到服务器

服务器：
1. 验证技能合法性
2. 执行技能
3. 返回确认/拒绝

客户端：
- 确认：预测正确，保持状态
- 拒绝：回滚预测结果
```

---

## 三、高级原理

### 10. GameplayEffectContext怎么扩展？为什么要手写NetSerialize？

**扩展目的**：传递自定义数据（暴击、伤害类型、击退方向等）

**实现步骤**：

1. **继承FGameplayEffectContext**：
```cpp
USTRUCT()
struct FMyGameplayEffectContext : public FGameplayEffectContext
{
    GENERATED_BODY()

public:
    bool bIsCriticalHit = false;
    bool bIsBlockedHit = false;
    FGameplayTag DamageType;

    virtual UScriptStruct* GetScriptStruct() const override;
    virtual FGameplayEffectContext* Duplicate() const override;
    virtual bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess) override;
    
    bool operator==(const FMyGameplayEffectContext& Other) const
    {
        return bIsCriticalHit == Other.bIsCriticalHit
            && bIsBlockedHit == Other.bIsBlockedHit;
    }
};
```

2. **实现NetSerialize**：
```cpp
bool FMyGameplayEffectContext::NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    Super::NetSerialize(Ar, Map, bOutSuccess);

    uint8 RepBits = 0;
    if (Ar.IsSaving())
    {
        if (bIsCriticalHit) RepBits |= 1 << 0;
        if (bIsBlockedHit)  RepBits |= 1 << 1;
    }

    Ar.SerializeBits(&RepBits, 2);

    if (Ar.IsLoading())
    {
        bIsCriticalHit = (RepBits & (1 << 0)) != 0;
        bIsBlockedHit  = (RepBits & (1 << 1)) != 0;
    }

    DamageType.NetSerialize(Ar, Map, bOutSuccess);

    bOutSuccess = true;
    return true;
}
```

3. **注册Traits**：
```cpp
template<>
struct TStructOpsTypeTraits<FMyGameplayEffectContext> 
    : public TStructOpsTypeTraitsBase2<FMyGameplayEffectContext>
{
    enum
    {
        WithNetSerializer = true,
        WithCopy = true,
    };
};
```

4. **重写AbilitySystemGlobals**：
```cpp
UCLASS()
class UMyAbilitySystemGlobals : public UAbilitySystemGlobals
{
    GENERATED_BODY()
public:
    virtual FGameplayEffectContext* AllocGameplayEffectContext() const override
    {
        return new FMyGameplayEffectContext();
    }
};
```

5. **配置ini**：
```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
AbilitySystemGlobalsClassName="/Script/YourProject.MyAbilitySystemGlobals"
```

**为什么手写NetSerialize**：
- Context是USTRUCT不是UObject，无法用UPROPERTY自动同步
- Context通过共享指针持有，支持多态，引擎无法自动处理
- 必须手动告诉引擎如何序列化子类新增字段

---

### 11. TStructOpsTypeTraits的原理是什么？

**本质**：C++模板特化实现的编译期类型查询表

**机制**：
```cpp
// 引擎默认模板
template<typename T>
struct TStructOpsTypeTraits {
    enum { WithNetSerializer = false };  // 默认不支持
};

// 你的特化
template<>
struct TStructOpsTypeTraits<FMyGameplayEffectContext> {
    enum { WithNetSerializer = true };   // 声明支持
};

// 引擎使用时
if constexpr (TStructOpsTypeTraits<T>::WithNetSerializer)
    Struct->NetSerialize(...);  // 有自定义，调用它
else
    DefaultSerialize(...);      // 没有，用默认
```

**为什么用模板特化而非虚函数**：
- USTRUCT无虚表，无法用虚函数
- 编译期确定，零运行时开销
- 可内联优化

---

### 12. ExecutionCalculation和ModifierMagnitudeCalculation区别？

| | ExecutionCalculation | MMC |
|---|---------------------|-----|
| 复杂度 | 高，可访问多个Attribute | 低，计算单个Modifier |
| 输出 | 可修改多个Attribute | 返回单个float |
| 访问 | Source和Target的Attribute | 有限访问 |
| 用途 | 复杂伤害公式 | 简单数值计算 |
| Prediction | 不支持 | 支持 |

**选择原则**：
- 简单倍率计算 → MMC
- 需要读取攻防、暴击等多属性 → Execution

---

### 13. 如何优化大量Actor的ASC性能？

**MinimalReplicationMode**：
```cpp
// 对于非玩家控制的Actor
ASC->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);
// 只同步Tag和Cue，不同步完整GE
```

**复制模式对比**：
| 模式 | 同步内容 | 适用 |
|------|----------|------|
| Full | 所有GE | 玩家角色 |
| Mixed | GE只发给Owner | 多人玩家 |
| Minimal | 只Tag和Cue | AI、NPC |

**其他优化**：
- 减少不必要的Attribute
- 合并频繁触发的GE
- 使用对象池复用GE Spec

---

### 14. AttributeSet能运行时动态增删吗？

**理论上可以**，但**强烈不推荐**。

**问题**：
- 已应用的GE可能引用被删除的Attribute
- 网络同步复杂度增加
- Prediction可能出问题

**推荐做法**：
- 预定义所有可能的AttributeSet
- 不需要的Attribute保持默认值
- 用多个AttributeSet组合而非动态增删

---

## 四、架构设计

### 15. GAS的优缺点？什么场景不适合用？

**优点**：
- 数据驱动，策划友好
- 网络同步内置支持
- 模块化，组件可复用
- 官方维护，Lyra/Fortnite验证

**缺点**：
- 学习曲线陡峭
- 简单需求过于重量级
- 调试困难
- 文档不完善

**不适合场景**：
- 单机简单游戏（杀鸡用牛刀）
- 技能系统极简的项目
- 团队无UE经验且工期紧

---

### 16. 如果让你重新设计GAS，会怎么改？

**问题1**：Context扩展太麻烦

**改进**：
```cpp
// 用组合替代继承
USTRUCT()
struct FGameplayEffectContext
{
    // 固定字段
    TWeakObjectPtr<AActor> Instigator;
    FHitResult HitResult;
    
    // 扩展数据容器
    TMap<FGameplayTag, FInstancedStruct> CustomData;
};
```

**问题2**：USTRUCT手写序列化

**改进**：
```cpp
// 新增标记，自动生成序列化代码
USTRUCT(NetSerializable)
struct FMyContext : public FGameplayEffectContext
{
    UPROPERTY(Replicated)
    bool bIsCriticalHit;
};
```

**问题3**：全局只能一个Context子类

**改进**：类型注册表，支持多种Context并存

---

### 17. 如何组织大量技能的配置和管理？

**目录结构**：
```
Abilities/
├── Base/
│   ├── GA_MeleeBase
│   └── GA_RangedBase
├── Character/
│   ├── GA_Slash
│   └── GA_Fireball
└── Enemy/
    └── GA_EnemyAttack

Effects/
├── Damage/
│   ├── GE_PhysicalDamage
│   └── GE_MagicDamage
├── Buff/
└── Debuff/
```

**管理策略**：
- 基类封装通用逻辑
- DataAsset配置技能参数
- GameplayTag分类索引
- 蓝图暴露策划可调参数

---

## 五、代码实战

### 18. 写一个ExecutionCalculation计算伤害

```cpp
UCLASS()
class UDamageExecution : public UGameplayEffectExecutionCalculation
{
    GENERATED_BODY()

public:
    UDamageExecution()
    {
        // 声明需要捕获的Attribute
        RelevantAttributesToCapture.Add(FGameplayEffectAttributeCaptureDefinition(
            UMyAttributeSet::GetAttackAttribute(),
            EGameplayEffectAttributeCaptureSource::Source,
            true));
            
        RelevantAttributesToCapture.Add(FGameplayEffectAttributeCaptureDefinition(
            UMyAttributeSet::GetDefenseAttribute(),
            EGameplayEffectAttributeCaptureSource::Target,
            true));
    }

    virtual void Execute_Implementation(
        const FGameplayEffectCustomExecutionParameters& Params,
        FGameplayEffectCustomExecutionOutput& OutParams) const override
    {
        float Attack = 0.f;
        float Defense = 0.f;
        
        // 获取Attribute值
        Params.AttemptCalculateCapturedAttributeMagnitude(AttackDef, FAggregatorEvaluateParameters(), Attack);
        Params.AttemptCalculateCapturedAttributeMagnitude(DefenseDef, FAggregatorEvaluateParameters(), Defense);
        
        // 计算伤害
        float Damage = FMath::Max(0.f, Attack - Defense);
        
        // 检查暴击
        const FGameplayEffectContextHandle& Context = Params.GetOwningSpec().GetContext();
        if (const FMyGameplayEffectContext* MyContext = 
            static_cast<const FMyGameplayEffectContext*>(Context.Get()))
        {
            if (MyContext->bIsCriticalHit)
            {
                Damage *= 2.0f;
            }
        }
        
        // 输出到MetaAttribute
        OutParams.AddOutputModifier(
            FGameplayModifierEvaluatedData(
                UMyAttributeSet::GetIncomingDamageAttribute(),
                EGameplayModOp::Additive,
                Damage));
    }
};
```

---

### 19. 写一个蓄力释放的AbilityTask

```cpp
UCLASS()
class UAbilityTask_ChargeAndRelease : public UAbilityTask
{
    GENERATED_BODY()

public:
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnChargeComplete, float, ChargeTime);
    
    UPROPERTY(BlueprintAssignable)
    FOnChargeComplete OnReleased;
    
    UPROPERTY(BlueprintAssignable)
    FOnChargeComplete OnMaxCharge;

    UFUNCTION(BlueprintCallable, Category = "Ability|Tasks", 
        meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility"))
    static UAbilityTask_ChargeAndRelease* CreateChargeTask(
        UGameplayAbility* OwningAbility,
        float MaxChargeTime);

private:
    float MaxChargeTime = 2.0f;
    float CurrentChargeTime = 0.f;
    bool bIsCharging = true;

    virtual void TickTask(float DeltaTime) override
    {
        if (!bIsCharging) return;
        
        CurrentChargeTime += DeltaTime;
        
        if (CurrentChargeTime >= MaxChargeTime)
        {
            bIsCharging = false;
            OnMaxCharge.Broadcast(MaxChargeTime);
            EndTask();
        }
    }

    // 绑定输入释放
    void OnInputReleased()
    {
        bIsCharging = false;
        OnReleased.Broadcast(CurrentChargeTime);
        EndTask();
    }
};
```

---

## 六、快速参考

### GAS核心类图

```
UAbilitySystemComponent
├── TArray<FGameplayAbilitySpec> ActivatableAbilities
├── FActiveGameplayEffectsContainer ActiveGameplayEffects
├── UAttributeSet* SpawnedAttributes
└── FGameplayTagCountContainer GameplayTagCountContainer

UGameplayAbility
├── FGameplayAbilitySpecHandle CurrentSpecHandle
├── UGameplayEffect* CostGameplayEffectClass
├── UGameplayEffect* CooldownGameplayEffectClass
└── TArray<UAbilityTask*> ActiveTasks

FGameplayEffectSpec
├── FGameplayEffectContextHandle EffectContext
├── TArray<FGameplayEffectModifiedAttribute> ModifiedAttributes
└── FGameplayTagContainer DynamicGrantedTags
```

### 常用宏和配置

```ini
# DefaultGame.ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
AbilitySystemGlobalsClassName="/Script/MyProject.MyAbilitySystemGlobals"
```

```cpp
// 常用Include
#include "AbilitySystemComponent.h"
#include "GameplayEffectTypes.h"
#include "AbilitySystemGlobals.h"
#include "GameplayEffectExecutionCalculation.h"
```

---

*文档版本：1.0*  
*最后更新：2026-03-27*