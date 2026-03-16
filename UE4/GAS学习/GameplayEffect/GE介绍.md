## GameplayEffect概览
***
### GE类型
GameplayEffect(GE)拥有三种类型，分别是即刻(Instant)、持续(Duration)和无限(Infinite).
```cpp
/** Gameplay effect duration policies */
UENUM()
enum class EGameplayEffectDurationType : uint8
{
	/** This effect applies instantly */
	Instant,
	/** This effect lasts forever */
	Infinite,
	/** The duration of this effect will be specified by a magnitude */
	HasDuration
};

// UGameplayEffect中对于三种类型的定义
/** Policy for the duration of this effect */
UPROPERTY(EditDefaultsOnly, Category=GameplayEffect)
EGameplayEffectDurationType DurationPolicy;
```

### GameplayEffect 与 GameplayEffectSpec
前者是一个GE的定义，后者是将效果添加到对象身上后，创建的GE示例.
GE实例上记录了:持续时间、层数、Source/Target上被捕获的GameplayTags，Modifiers实例

### 施加GE
GE可以通过GA或者ASC来添加：
1. GA添加GE,可以使用`UGameplayAbility::ApplyGameplayEffectSpecToOwner`
2. ASC添加GE，可以使用`UAbilitySystemComponent::ApplyGameplayEffectToSelf`
最终都会调用到`UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf`

### Instant与Duration类型激活流程
TODO