## 前言

做了前面的任务后，我们可以实现回血，回蓝，掉血等功能，但是我们发现一个问题，这些东西没有被限制，当血量低于0时会继续掉，大于Max值也会继续升，这令人很烦恼。为了限制他们，我们可以使用Clamp函数来限制它。但什么时候限制它呢？AS提供了一个PreAttributeChange()，可以让我们在属性变化前就对属性做出限制；

## PreAttributeChange

```
.h
virtual void PreAttributeChange(const FGameplayAttribute &Attribute,float &NewValue) override;
AttributeSet.cpp
void UAuraAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
	Super::PreAttributeBaseChange(Attribute, NewValue);

	if (Attribute == GetHealthAttribute()) {

		NewValue = FMath::Clamp(NewValue, 0, GetMaxHealth());
	}
	if (Attribute == GetManaAttribute()) {
		NewValue = FMath::Clamp(NewValue, 0, GetMaxMana()); 
	}
}
```

这个函数只要在属性发生变化时就会被调用；

在 `PreAttributeChange` 函数中，属性的约束会在后续操作重新计算当前值时失效。具体来说，以下几点详细说明了这一过程：

### 例子

假设有一个游戏属性“健康值”，其当前值（CurrentValue）为100，一个修饰符增加了20，一个游戏效果减少了10。

1. **`PreAttributeChange` 执行** ：

* 假设这个钩子函数在修改前对健康值进行了检查，确保它不会超过某个上限，例如150。

1. **属性变化触发** ：

* 修改后的健康值将是 100（当前值）+ 20（修饰符）- 10（游戏效果） = 110。

1. **后续操作重新计算** ：

* 如果有另一个游戏效果或操作再次改变健康值，系统会重新计算这个值，临时约束（比如检查是否超过150）需要再次应用，否则就会失效。

## PostGameplayEffectExecute

`PostGameplayEffectExecute(const FGameplayEffectModCallbackData & Data)`仅在 `即刻(Instant)GameplayEffect`对 `Attribute`的BaseValue修改之后触发.也就是说这个函数写一些属性更改后触发的逻辑；

```
UAuraAttributeSet.cpp
void UAuraAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
	Super::PostGameplayEffectExecute(Data);

	FEffectProperties Props;
	SetEffectProperties(Data, Props);
}


struct FEffectProperties
{
	GENERATED_BODY();


	UPROPERTY()
	UAbilitySystemComponent* SourceASC=nullptr;
	UPROPERTY()
	UAbilitySystemComponent* TargetASC=nullptr;

	UPROPERTY()
	AActor* SourceAvatarActor = nullptr;
	UPROPERTY()
	AActor* TargetAvatarActor = nullptr;

	UPROPERTY()
	AController* TargetController = nullptr;
	UPROPERTY()
	AController* SourceController = nullptr;

	UPROPERTY()
	ACharacter* TargetCharacter = nullptr;
	UPROPERTY()
	ACharacter* SourceCharacter = nullptr;

	FGameplayEffectContextHandle EffectContextHandle;
};


void UAuraAttributeSet::SetEffectProperties(const FGameplayEffectModCallbackData& Data, FEffectProperties& EffectProperties)
{
	//Source = Causer of the effect,Target =Target of the effect（owner of this AS)
	EffectProperties.EffectContextHandle = Data.EffectSpec.GetEffectContext();
	EffectProperties.SourceASC = EffectProperties.EffectContextHandle.GetInstigatorAbilitySystemComponent();

	if (IsValid(EffectProperties.SourceASC) && EffectProperties.SourceASC->AbilityActorInfo.IsValid() && EffectProperties.SourceASC->AbilityActorInfo->AvatarActor.IsValid()) {
		EffectProperties.SourceAvatarActor = EffectProperties.SourceASC->AbilityActorInfo->AvatarActor.Get();
		EffectProperties.SourceController = EffectProperties.SourceASC->AbilityActorInfo->PlayerController.Get();
		if (EffectProperties.SourceController == nullptr && EffectProperties.SourceAvatarActor != nullptr)
		{
			if (const APawn* Pawn = Cast<APawn>(EffectProperties.SourceAvatarActor)) {
				EffectProperties.SourceController = Pawn->GetController();
			}
		}
		if (EffectProperties.SourceController) {
			EffectProperties.SourceCharacter = Cast<ACharacter>(EffectProperties.SourceController->GetPawn());
		}
	}

	//拿到Target的Avatar，Controller,Character,ASC;
	if (Data.Target.AbilityActorInfo.IsValid() && Data.Target.AbilityActorInfo->AvatarActor.IsValid()) {
		EffectProperties.TargetAvatarActor = Data.Target.AbilityActorInfo->AvatarActor.Get();
		EffectProperties.TargetController = Data.Target.AbilityActorInfo->PlayerController.Get();
		EffectProperties. TargetCharacter = Cast<ACharacter>(EffectProperties.TargetAvatarActor);
		EffectProperties.TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(EffectProperties.TargetAvatarActor);
	}
}
```

当该函数发生变化的时候，我们将source和Target的数据放到EffectProperties里面，目前我们做到这里，可能后面还需要在Post函数里添加些逻辑，而这些逻辑可能需要这些数据之类的；
