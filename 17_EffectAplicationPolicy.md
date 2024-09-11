## 前言

现在我们将效果应用到actor上大概是这样的流程，在Sphere经过重叠后立即调用ApplyEffectToActor,然后就看是否有需要执行Destory();但这样并不是很灵活。我们为了更加的模块化，将apply和Removal做了个枚举，这样在以后我们需要添加Apply环境的时候会更加方便

```
UENUM(BlueprintType) 
enum class EEffectApplicationPolicy :uint8
{
	ApplyOnOverlap,
	ApplyOnEndOverlap,
	DoNotApply
};

UENUM(BlueprintType)
enum class EEffectRemovalPolicy :uint8
{
	RemoveOnEndOverlap,
	DoNotRemove
};
```

可以看到Effect在Overlap和EndOverlap下Apply，以后倘若需要其他情况下Apply,就可以更灵活的处理了；

## 声明

我们有三种DurationPolicy,所以我们给每一种Policy声明一个枚举，通过枚举类型决定是否将Effect应用到Actor上面；

```
	UPROPERTY(EditAnywhere,BlueprintReadOnly,Category="Effects")
	TSubclassOf<UGameplayEffect>InstantGameplayEffectClass;

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Effects")
	EEffectApplicationPolicy InstantEffectApplicationPolicy= EEffectApplicationPolicy::DoNotApply;

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Effects")
	TSubclassOf<UGameplayEffect>DurationGameplayEffectClass;

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Effects")
	EEffectApplicationPolicy DurationEffectApplicationPolicy=EEffectApplicationPolicy::DoNotApply;

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Effects")
	TSubclassOf<UGameplayEffect>InfiniteGameplayEffectClass;

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Effects")
	EEffectApplicationPolicy InfiniteEffectApplicationPolicy=EEffectApplicationPolicy::DoNotApply;

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Effects")
	EEffectRemovalPolicy InfiniteEffectRemovalPolicy=EEffectRemovalPolicy::RemoveOnEndOverlap;
```

有了枚举类型，我们要写函数，这个函数根据枚举类型来决定是否ApplyEffect;因为前面提到Apply的情况分为两种，所以我们可以写两个函数。值得一提的是移除Effect，我们只在infinitePolicy下，所以只在InfiniteGameplayEffectClass下面写了关于何时移除的枚举；

```
.h
	UFUNCTION(BlueprintCallable)
	void OnOverlap(AActor *TargetActor);

	UFUNCTION(BlueprintCallable)
	void OnEndOverlap(AActor *TargetActor);

.cpp
void AAuraEffectActor::OnOverlap(AActor* TargetActor)
{
	if (InstantEffectApplicationPolicy == EEffectApplicationPolicy::ApplyOnOverlap) {
		ApplyEffectToActor(TargetActor, InstantGameplayEffectClass);
	}
	if (DurationEffectApplicationPolicy == EEffectApplicationPolicy::ApplyOnOverlap) {
		ApplyEffectToActor(TargetActor, DurationGameplayEffectClass);
	}
	if (InfiniteEffectApplicationPolicy == EEffectApplicationPolicy::ApplyOnOverlap) {
		ApplyEffectToActor(TargetActor, InfiniteGameplayEffectClass);
	}
}

void AAuraEffectActor::OnEndOverlap(AActor* TargetActor)
{
	if (InstantEffectApplicationPolicy == EEffectApplicationPolicy::ApplyOnOverlap) {
		ApplyEffectToActor(TargetActor, InstantGameplayEffectClass);
	}
	if (DurationEffectApplicationPolicy == EEffectApplicationPolicy::ApplyOnEndOverlap) {
		ApplyEffectToActor(TargetActor, DurationGameplayEffectClass);
	}
	if (InfiniteEffectApplicationPolicy == EEffectApplicationPolicy::ApplyOnEndOverlap) {
		ApplyEffectToActor(TargetActor, InfiniteGameplayEffectClass);
	}
	if (InfiniteEffectRemovalPolicy == EEffectRemovalPolicy::RemoveOnEndOverlap) {
		UAbilitySystemComponent* TargetASC = UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(TargetActor);
		if (!IsValid(TargetASC)) return;

		TArray<FActiveGameplayEffectHandle>HandleToRemove;
		for (auto HandlePair : ActiveEffectHandles) {
			if (TargetASC == HandlePair.Value) {
				TargetASC->RemoveActiveGameplayEffect(HandlePair.Key,1);
				HandleToRemove.Add(HandlePair.Key);
				//ActiveEffectHandles.FindAndRemoveChecked(HandlePair.Key);
			}
		}
		for (auto& Handle : HandleToRemove) {
			ActiveEffectHandles.FindAndRemoveChecked(Handle);
		}
	}
}
```

我们可以看到EndOverlap中的代码长，那是因为我们将移除Effect的逻辑写在这里面了；当然可以在外面专门写一个关于移除逻辑的函数，然后在这里调用。

可以看到最后一个if语句是移除Effect,但是发现缺了很多东西，如下；

```
AuraEffectActor.h
TMap<FActiveGameplayEffectHandle, UAbilitySystemComponent*> ActiveEffectHandles;
AuraEffectActor.cpp
void AAuraEffectActor::ApplyEffectToActor(AActor* TargetActor, TSubclassOf<UGameplayEffect> GameplayEffectClass)
{
	UAbilitySystemComponent* TargetASC=UAbilitySystemBlueprintLibrary::GetAbilitySystemComponent(TargetActor);
	if (TargetASC == nullptr) return;

	//将GamePlayEffect应用到ASC上面；
	check(GameplayEffectClass);
	FGameplayEffectContextHandle GameplayEffectContextHandle=TargetASC->MakeEffectContext();
	GameplayEffectContextHandle.AddSourceObject(this);
	FGameplayEffectSpecHandle EffectSpecHandle=TargetASC->MakeOutgoingSpec(GameplayEffectClass, ActorLevel, GameplayEffectContextHandle);
	const FActiveGameplayEffectHandle ActiveEffectHandle=TargetASC->ApplyGameplayEffectSpecToSelf(*EffectSpecHandle.Data.Get());

	const bool bIsInfinite=EffectSpecHandle.Data.Get()->Def.Get()->DurationPolicy == EGameplayEffectDurationType::Infinite;
	if (bIsInfinite && InfiniteEffectRemovalPolicy==EEffectRemovalPolicy::RemoveOnEndOverlap) {
		ActiveEffectHandles.Add(ActiveEffectHandle,TargetASC);
	}
}
```

可以看到我们加了一个map,并且对ApplyEffectToActor（）的逻辑进行了补充，这全是为了实现移除效果；至于为什么这么做，看代码就可以了；
