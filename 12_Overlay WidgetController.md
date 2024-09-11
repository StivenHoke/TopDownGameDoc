## 前言

这一节把整体的架构整清楚了，在HUD里面，我们将Model的数据传给了WidgetController，然后给Widget设置了对应的Controller,因为采用的是overlay的设计，小的控件放在了一个overlay上面，所以我们可以设置一个overlayWidget和对应的OverlayWidgetController就可以了；

## AuraWigetController

在之前我们在这个类里面设置了4个变量，这些变量用来获取Model的数据，我们需要对它进行初始化	，但对他进行初始化前我们需要考虑一个问题，我们只希望有一个整体的UI，也就是单例模式，所以我们不提供构造函数，所以这个类的初始化我们希望通过自定义的函数进行设置，并且对于这种多变量的初始化，我们一般可以设置一个结构体，对他进行初始化。

```
USTRUCT(BlueprintType)
struct FWidgetControllerParams
{
	GENERATED_BODY()
	FWidgetControllerParams() {};
	FWidgetControllerParams(APlayerController *PC,APlayerState *PS,UAbilitySystemComponent *ASC,UAttributeSet *AS) 
		:PlayerController(PC),PlayerState(PS),AbilitySystemComponent(ASC),AttributeSet(AS){};

	UPROPERTY(EditAnywhere,BlueprintReadWrite)
	TObjectPtr<APlayerController>PlayerController = nullptr;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	TObjectPtr<APlayerState>PlayerState = nullptr;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	TObjectPtr<UAbilitySystemComponent>AbilitySystemComponent = nullptr;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	TObjectPtr<UAttributeSet>AttributeSet = nullptr;

};
```

结构体有了，然后一个赋值函数	

```
void UAuraWidgetController::SetWidgetControllerParams(const FWidgetControllerParams& WCParams)
{
	PlayerController = WCParams.PlayerController;
	PlayerState = WCParams.PlayerState;
	AbilitySystemCOmponent = WCParams.AbilitySystemComponent;
	AttributeSet = WCParams.AttributeSet;
}
```

## OverlayWidgetController

我们会有很多的WidgetController，所以不希望只有一个AuraWidgetController（继承自UObject),对应的OverlayWidget的OverlayWidgetController就诞生了，我们希望能在对应的层级上调整它。

目前我们只有这个类就够了，他继承的数据够我们使用；

## AuraHUD

在HUD中我们希望能对这两个类进行实例化，实例化后进行连接，就用我们定义的函数；

定义Widget的指针和WidgetController的指针；

```
TObjectPtr<UAuraUserWidget> OverlayWidget;

TObjectPtr<UOverlayWidgetController>OverlayWidgetController;

```

我们希望能在这个类里面对他们进行实例化的方法，对于OverlayWidgetController

```
UOverlayWidgetController* AAuraHUD::GetOverlayWidgetController(const FWidgetControllerParams& WCParams)
{
	if (OverlayWidgetController == nullptr) {
		OverlayWidgetController = NewObject<UOverlayWidgetController>(this, OverlayWidgetControllerclass);
		OverlayWidgetController->SetWidgetControllerParams(WCParams);

		return OverlayWidgetController;
	}
	return OverlayWidgetController;
}
```

我们希望他是单例，所以有则返回实例，无则创建一个，由于WidgetController是继承自UObejct,所以它的创建是通过NewObject<>();对于第二个参数需要使用TSubclassOf,根据类来创建实例。

```
TSubclassOf<OverlayWidgetController>OverlayWidgetControllerclass;
```

我们至此发现，UE中原生类的实例的创建都有封装的方法，像UserWidget有CreateWidget<>()；

## 第一动力

我们前面写了这么多方法，初始化的方法，实例的方法，但是都需要调用，需要真正的参数传递进来不是嘛；在这里我们的第一动力是那4个数据参数，有了4个数据实例的传入，我们的实例都将被真实创建，所以请看下面；

```
void AAuraHUD::InitOverlay(APlayerController* PC, APlayerState* PS, UAbilitySystemComponent* ASC, UAttributeSet* AS)
{
	checkf(OverlayWidgetClass, TEXT("OverlayWidgetClass uninitialized,please fill out BP_AuraHUD "));
	checkf(OverlayWidgetControllerclass, TEXT("OverlayWidgetControllerClass uninitialized,Please fill out BP_AuraHUD"));

	const FWidgetControllerParams WidgetControllerParams(PC, PS, ASC, AS);
	UOverlayWidgetController* WidgetController = GetOverlayWidgetController(WidgetControllerParams);


	UUserWidget* Widget = CreateWidget<UUserWidget>(GetWorld(), OverlayWidgetClass);
	OverlayWidget = Cast<UAuraUserWidget>(Widget);
	OverlayWidget->SetWidgetController(WidgetController);
	Widget->AddToViewport();
	
}
```

我们只需要这个函数被调用，那么我们的UI架构就能起来了，所以在哪里被调用呢？被调用的地方应该有这四个参数，且这个函数能被执行。

前面说过ASC需要知道他被谁拥有，InitAbilityActorInfo()函数，这个函数写在Character里的ProcessBy()和OnRep_PlayerState()上面，所以这里就是我们可以调用这个第一动力的地方嘛；

```
void AAuraCharacter::InitAbilityActorInfo()
{
	AAruaPlayerState* AuraPlayerState = GetPlayerState<AAruaPlayerState>();
	check(AuraPlayerState);
	AuraPlayerState->GetAbilitySystemComponent()->InitAbilityActorInfo(AuraPlayerState, this);
	//自身的ASC和AS接受到了服务器的ASC和AS;
	AbilitySystemComponent = AuraPlayerState->GetAbilitySystemComponent();
	AttributeSet = AuraPlayerState->GetAttributeSet();

#	if (AAuraPlayerController* AuraPlayerController = Cast<AAuraPlayerController>(GetController()))
	{
		if (AAuraHUD* AuraHUD = Cast<AAuraHUD>(AuraPlayerController->GetHUD())) {
			AuraHUD->InitOverlay(AuraPlayerController, AuraPlayerState, AbilitySystemComponent, AttributeSet);
		}
	}
}
```

这一节的内容从#开始，因为InitOverlay写在HUD里面，所以我们经过两层寻找；
