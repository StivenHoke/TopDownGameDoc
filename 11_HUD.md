## 前言

我们设置UI一般都是基于蓝图来设计的，蓝图继承自UserWidget,但是我们上一节做了一个继承自UserWidget的UAuraUserWidget，它相比于父类添加了和Controller之间的关系；但是其他没有变，所以我们设计具体的UI，应该继承UAuraUserWidget;

问题来了，我们设计好WBP后该往哪里放呢？放在关卡蓝图中，切到下一个关卡，UI直接没有了，不太合适，所以我们应该放在GameMode里面HUD类嘛；

## New class

既然我们要用HUD来播放我们的WBP,那我们肯定要用到TSubclassOf了，这样就可以在BP_HUD里面挑选具体的想播放的WBP了，还是很简单的；

.h

```
	UPROPERTY()
	TObjectPtr<UAuraUserWidget> OverlayWidget;

protected:
	virtual void BeginPlay() override;
private:
	UPROPERTY(EditAnywhere)
	TSubclassOf<UAuraUserWidget> OverlayWidgetClass;
```

.cpp

```
void AAuraHUD::BeginPlay()
{
	Super::BeginPlay();

	UUserWidget* Widget=CreateWidget<UUserWidget>(GetWorld(), OverlayWidgetClass);
	Widget->AddToViewport();
}
```
