## AttributeSet

接下来，为角色添加属性就在这里添加了，它里面的所有数据类型均为float，FGamePlayAttributeData 这个结构体就封装了两个浮点数类型；

就是按照步骤添加就好了，没什么好讲的；

### 声明变量

```
	UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_Health ,Category="vital Attribute")
	FGameplayAttributeData Health;
```

**ReplicatedUsing** ：这个参数指定当这个属性从服务器复制到客户端时要调用的函数。在这个例子中，当 `Health`属性在客户端更新时，将调用 `OnRep_Health`函数，这个函数是自己定义的；

### 触发函数

```
	UFUNCTION()
	void OnRep_Health(const FGameplayAttributeData &OldHealth) const
	{
		GAMEPLAYATTRIBUTE_REPNOTIFY(UAuraAttributeSet, Health, OldHealth);
	}

```

OldHealth是属性未更新前的值；

GAMEPLAYATTRIBUTE_REPNOTIFY(UAuraAttributeSet, Health, OldHealth);

1. **属性值变化检查和更新** ：

* 宏会检查 `Health` 的新值与 `OldHealth` 是否不同，如果不同则更新 `OldHealth` 为当前的 `Health` 值。

1. **通知系统属性已更新** ：

* 宏会确保调用通知系统，以便其他依赖于 `Health` 值的系统可以相应地进行更新。

### 复制变量声明

```
void UAuraAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, Health, COND_None, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, Mana, COND_None, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(UAuraAttributeSet, MaxMana, COND_None, REPNOTIFY_Always);
}
```

这是一个虚函数，用于设置类中哪些属性需要进行网络复制。在继承类中重写这个函数，以便告诉引擎哪些属性需要同步。
