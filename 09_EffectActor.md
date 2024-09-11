## 前言

EffectActor是对actor的属性更改；

对于EffectActor会用到GameplayEffect,但是目前我们并没有学习到它，所以用另一种方法来进行对属性的更改。这种更改是基于碰撞体进行广播后类型检查实现的

## New class

创建一个基于actor的类，将用不到的Tick函数和一些注释删除;

EffectActor我们希望能做的是改变属性，比如吃药回血等效果，所以需要将他放入场景，并添加一个球体组件；

```
	UPROPERTY(VisibleAnywhere)
	TObjectPtr<USphereComponent>Sphere;

	UPROPERTY(VisibleAnywhere)
	TObjectPtr<UStaticMeshComponent>Mesh;
```

然后对它进行初始化；

```
	Mesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("StaticMesh"));
	SetRootComponent(Mesh);
	Sphere = CreateDefaultSubobject<USphereComponent>(TEXT("SphereCollision"));
	Sphere->SetupAttachment(Mesh);
```

## 广播

我们在蓝图的经验可知，到我们进入一个collision区域，会触发一些逻辑对嘛；比如蓝图中的开关门教程，当我们进入区域，可以开门，离开区域关门。

所以我们在BeginPlay函数写了	这种逻辑

```
void AAuraEffectActor::BeginPlay()
{
	Super::BeginPlay();

	Sphere->OnComponentBeginOverlap.AddDynamic(this, &AAuraEffectActor::OnOverlap);
	Sphere->OnComponentEndOverlap.AddDynamic(this, &AAuraEffectActor::EndOverlap);
}
```

进入球体区域会触发后面两个自定义的函数，不过这两个函数的参数需要ctrl+左键进入PrimitiveComponent中去看；

## 类型检查

进入Sphere的区域我们就实现自定义的函数，这个函数实现了改变我们UAuraAttributeSet里面的属性；

```
void AAuraEffectActor::OnOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	//TODO：Change this  to apply GameplayEffect.For now,using const_const as a hack1
	if (IAbilitySystemInterface* ASCInterface = Cast<IAbilitySystemInterface>(OtherActor)) {
		const UAuraAttributeSet* AuraAttributeSet=Cast<UAuraAttributeSet>(ASCInterface->GetAbilitySystemComponent()->GetAttributeSet(UAuraAttributeSet::StaticClass()));
		UAuraAttributeSet* MutableAttributeSet = const_cast<UAuraAttributeSet*>(AuraAttributeSet);
		MutableAttributeSet->SetHealth(AuraAttributeSet->GetHealth() + 50);
		Destroy();
	}
}
```

大概思路就是检查我们进入这个区域的actor有没有ASI接口类，有了这个接口类我们就可以返回这个actor的ASC，ASC拿到我们的属性集，我们的属性集里有Attribute Accessors可以对属性进行更改和获取；
