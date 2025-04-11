# GameplayAbility

## GameplayEffect

### 对象

#### FGameplayEffectSpec

用于描述 `GameplayEffect` 的 **运行时实例** 的核心数据结构

`UGameplayEffect` 保存着一系列配置信息，`FGameplayEffectSpec` 保存着运行时所需的信息以及原始的配置数据

| 属性名 | 作用 |  
| --- | --- |  
| Duration	|效果的持续时间（秒）。<br>- INSTANT_APPLICATION 表示瞬时效果<br>- INFINITE_DURATION 表示无限持续效果 | 
| Period|周期性效果的触发间隔（秒）。<br>- NO_PERIOD 表示非周期性效果 | 
| ChanceToApplyToTarget | 效果应用到目标的概率（0~1）。<br>（已弃用，由 UChanceToApplyGameplayEffectComponent 替代） | 
| CapturedSourceTags | 捕获来源（Source）的标签（如施法者的状态标签）。<br>在 GameplayEffectSpec 创建时捕获 | 
| CapturedTargetTags | 捕获目标（Target）的标签（如目标的状态标签）。<br>在效果执行时捕获 | 
| DynamicGrantedTags | 动态授予的标签（非来自 UGameplayEffect 配置）。<br>用于临时状态（如“燃烧中”） | 
| DynamicAssetTags | 动态添加的资产标签（非来自 UGameplayEffect 配置）。<br>（已弃用，推荐使用 AddDynamicAssetTag 等方法） | 
| Modifiers | 存储所有修饰器的计算后数值（如 Damage = 100）。<br>每个元素对应 UGameplayEffect 中的一个 Modifier | 
| StackCount | 效果的堆叠次数（如中毒效果叠加层数）。<br>（已弃用，未来将改为私有） | 
| bCompletedSourceAttributeCapture | 标记是否已完成来源（Source）属性捕获（如施法者的攻击力） | 
| bCompletedTargetAttributeCapture | 标记是否已完成目标（Target）属性捕获（如目标的防御力） | 
| bDurationLocked | 标记持续时间是否被锁定（防止后续修改） | 
| GrantedAbilitySpecs | 效果授予的技能定义（如被动技能）。<br>（已弃用，由 AbilitiesGameplayEffectComponent 替代） | 
| SetByCallerNameMagnitudes	 | 通过 FName 动态设置的数值（如 SetByCallerMagnitude("Damage", 100)） | 
| SetByCallerTagMagnitudes | 通过 GameplayTag 动态设置的数值（如 SetByCallerMagnitude(DamageType.Fire, 50)） | 
| EffectContext | 效果的上下文信息（如施法者、目标、技能实例） | 
| Level  | 效果的等级（影响数值计算，如 Damage = BaseDamage × Level） | 

### 添加 GE 的流程

流程图在 [GE的添加流程](./GE的添加流程.drawio)

以 `BlueprintCallable` 的 `BP_ApplyGameplayEffectToSelf` 作为入口

1. 通过传入的 `TSubclassOf<UGameplayEffect>` 创建 GE 对象
2. 调用 `ApplyGameplayEffectToTarget`

```cpp
UGameplayEffect* GameplayEffect = GameplayEffectClass->GetDefaultObject<UGameplayEffect>();
return ApplyGameplayEffectToTarget(GameplayEffect, Target, Level, Context);	
```

> 注意这里是通过 `GetDefaultObject` 获取目标的 CDO 对象，也就是说所有 GE 在这里得到的 `GameplayEffect` 都是同一个对象，这为后面判断堆叠提供了帮助

在 `ApplyGameplayEffectToTarget` 中

1. 通过传入参数创建 `FGameplayEffectSpec` 
2. 调用 `ApplyGameplayEffectSpecToTarget`

```cpp
FGameplayEffectSpec	Spec(GameplayEffect, Context, Level);
return ApplyGameplayEffectSpecToTarget(Spec, Target, PredictionKey);
```

在 `ApplyGameplayEffectSpecToTarget` 中

1. 判断是否需要预测，以此来清空预测键 `PredictionKey` 
2. 调用 `Target->ApplyGameplayEffectSpecToSelf` 获得 GE 的 `Handle`

> 接下来所有代码都不考虑网络同步和预测的问题

在 `Target->ApplyGameplayEffectSpecToSelf` 中

1. 检查
   1. 通过 `GameplayEffectApplicationQueries` 检查当前要添加的 GE 是否有效
   2. 通过 `ActiveGameplayEffects` 和当前要添加的 GE 检查能否添加
      - 通过 `GEComponent` 来判断的
      - 例如：`AssetTagsGameplayEffectComponent`、`AbilitiesGameplayEffectComponent` 等 `GEComponent` 配置
   3. 检查当前准备添加的 `GE` 是否配置了有效的 `Attribute`
2. 判断是否是立即执行的 GE
   - 如果是**立即执行**的 GE：`ExecuteGameplayEffect(*OurCopyOfSpec, PredictionKey)` 直接执行 GE
   - 如果是**持续时间**的 GE：`ActiveGameplayEffects.ApplyGameplayEffectSpec` 添加 GE 到容器中

在 `ActiveGameplayEffects.ApplyGameplayEffectSpec` 中

1. 判断堆叠
   - 如果堆叠：更新叠层计数，根据条件刷新持续时间、重置计数器等操作，使用现有的 FActiveGameplayEffect 实例
   - 如果不堆叠：创建 FActiveGameplayEffect 实例
2. 重新计算 FActiveGameplayEffect 数值，收集其绑定的属性
3. 计算持续时间和周期，绑定 Timer 到 FActiveGameplayEffect 实例上
4. 根据是否堆叠触发不同的事件
   - 如果堆叠：触发 `OnStackCountChange`
   - 如果不堆叠：触发 `InternalOnActiveGameplayEffectAdded` 

在代码中有两个地方使用了 `TimeManager`

- 第一个 `Timer` 是在持续时间结束后触发，用于标记效果过期（可以移除效果），只触发一次所以 bLoop 设置为 false
- 第二个 `Timer` 是固定时间执行周期性效果（每秒造成1点伤害），需要循环触发所以设置为 true
  - 如果 GE 的 `bExecutePeriodicEffectOnApplication` 为 true，表示 GE 添加时立刻执行一次，所以会额外注册一个 `TimerManager.SetTimerForNextTick`

```cpp
// Calculate Duration mods if we have a real duration
if (DurationBaseValue > 0.f)
{
    float FinalDuration = AppliedEffectSpec.CalculateModifiedDuration();
    // 一些判断和特殊数据处理
    if (Owner && bSetDuration)
    {
        FTimerManager& TimerManager = Owner->GetWorld()->GetTimerManager();
        FTimerDelegate Delegate = FTimerDelegate::CreateUObject(Owner, &UAbilitySystemComponent::CheckDurationExpired, AppliedActiveGE->Handle);
        TimerManager.SetTimer(AppliedActiveGE->DurationHandle, Delegate, FinalDuration, false);
        if (!ensureMsgf(AppliedActiveGE->DurationHandle.IsValid(), TEXT("Invalid Duration Handle after attempting to set duration for GE %s @ %.2f"), 
            *AppliedActiveGE->GetDebugString(), FinalDuration))
        {
            // Force this off next frame
            TimerManager.SetTimerForNextTick(Delegate);
        }
    }
}

// Register period callbacks with the timer manager
if (bSetPeriod && Owner && (AppliedEffectSpec.GetPeriod() > UGameplayEffect::NO_PERIOD))
{
    FTimerManager& TimerManager = Owner->GetWorld()->GetTimerManager();
    FTimerDelegate Delegate = FTimerDelegate::CreateUObject(Owner, &UAbilitySystemComponent::ExecutePeriodicEffect, AppliedActiveGE->Handle);
        
    // The timer manager moves things from the pending list to the active list after checking the active list on the first tick so we need to execute here
    if (AppliedEffectSpec.Def->bExecutePeriodicEffectOnApplication)
    {
        TimerManager.SetTimerForNextTick(Delegate);
    }

    TimerManager.SetTimer(AppliedActiveGE->PeriodHandle, Delegate, AppliedEffectSpec.GetPeriod(), true);
}
```

在 `InternalOnActiveGameplayEffectAdded` 函数中

1. 会根据 GEComponent 来判断当前 GE 能否**被激活**或者说是否**被抑制**(`bIsInhibited`)
2. 调用 `UAbilitySystemComponent::InhibitActiveGameplayEffect`


在 `UAbilitySystemComponent::InhibitActiveGameplayEffect` 函数中

1. 根据 GE 是否被 **抑制**
   - 被抑制：执行 `RemoveActiveGameplayEffectGrantedTagsAndModifiers`
   - 不被抑制：执行 `AddActiveGameplayEffectGrantedTagsAndModifiers`
2. 触发 `OnInhibitionChanged` 事件
3. 在 `FScopedAggregatorOnDirtyBatch` 对象析构的时候触发
   - `ActiveGameplayEffects.OnMagnitudeDependencyChange` 属性变化事件
   - `OnDirty` 脏数据事件触发



### 执行 GE 的流程

```cpp
FTimerDelegate Delegate = FTimerDelegate::CreateUObject(Owner, &UAbilitySystemComponent::ExecutePeriodicEffect, AppliedActiveGE->Handle);
```

根据上面的代码，直接定位执行位置是 `UAbilitySystemComponent::ExecutePeriodicEffect`

通过函数调用，真正执行的代码在 `FActiveGameplayEffectsContainer::ExecuteActiveEffectsFrom` 函数中

1. 设置 `TargetTags` 即 Owner 标记上的 GameplayTags
2. 计算 GE 的 Modifiers
3. 通过循环调用 `InternalExecuteMod` 应用 `Modifier`
4. 遍历 `Executions` 数组，计算得到 `ConditionalEffectSpecs` 将要附加的 GE
5. 触发 GC
6. 应用 `ConditionalEffectSpecs`
7. 最后触发事件

重点在于两个函数 `CalculateModifierMagnitudes` 和 `InternalExecuteMod`

在 `CalculateModifierMagnitudes` 中

```cpp
const FGameplayModifierInfo& ModDef = Def->Modifiers[ModIdx];
FModifierSpec& ModSpec = Modifiers[ModIdx];

if (ModDef.ModifierMagnitude.AttemptCalculateMagnitude(*this, ModSpec.EvaluatedMagnitude) == false)
{
    ModSpec.EvaluatedMagnitude = 0.f;
    ABILITY_LOG(Warning, TEXT("Modifier on spec: %s was asked to CalculateMagnitude and failed, falling back to 0."), *ToSimpleString());
}
```

很清晰，通过在 GE 中配置的 `Def->Modifiers` 计算出 `ModSpec.EvaluatedMagnitude` 的值

```cpp
switch (MagnitudeCalculationType)
{
case EGameplayEffectMagnitudeCalculation::ScalableFloat:break;
case EGameplayEffectMagnitudeCalculation::AttributeBased:break;
case EGameplayEffectMagnitudeCalculation::CustomCalculationClass:break;
case EGameplayEffectMagnitudeCalculation::SetByCaller:break;
// ...
}
```

在 `AttemptCalculateMagnitude` 中通过枚举，来计算具体的值内容


在 `InternalExecuteMod` 函数中，核心代码如下

```cpp
if (AttributeSet->PreGameplayEffectExecute(ExecuteData))
{
    float OldValueOfProperty = Owner->GetNumericAttribute(ModEvalData.Attribute);
    ApplyModToAttribute(ModEvalData.Attribute, ModEvalData.ModifierOp, ModEvalData.Magnitude, &ExecuteData);

    FGameplayEffectModifiedAttribute* ModifiedAttribute = Spec.GetModifiedAttribute(ModEvalData.Attribute);
    if (!ModifiedAttribute)
    {
        // If we haven't already created a modified attribute holder, create it
        ModifiedAttribute = Spec.AddModifiedAttribute(ModEvalData.Attribute);
    }
    ModifiedAttribute->TotalMagnitude += ModEvalData.Magnitude;

    {
        SCOPE_CYCLE_COUNTER(STAT_PostGameplayEffectExecute);
        /** This should apply 'gamewide' rules. Such as clamping Health to MaxHealth or granting +3 health for every point of strength, etc */
        AttributeSet->PostGameplayEffectExecute(ExecuteData);
    }
}
```

基本流程也很简单

1. 调用 `PreGameplayEffectExecute` 判断能否触发
2. 
