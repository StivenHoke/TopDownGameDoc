## 前言

AttributeSet为我们提供了一些宏，来方便我们对FGameplayAttributeDate进行一些数据上的更改而不用自定义函数；简而言之就是我们使用它提前定义好的宏，我们就可以直接使用它的函数；

引擎中的注释如下

```
/**
 * This defines a set of helper functions for accessing and initializing attributes, to avoid having to manually write these functions.
 * It would creates the following functions, for attribute Health
 *
 *	static FGameplayAttribute UMyHealthSet::GetHealthAttribute();
 *	FORCEINLINE float UMyHealthSet::GetHealth() const;
 *	FORCEINLINE void UMyHealthSet::SetHealth(float NewVal);
 *	FORCEINLINE void UMyHealthSet::InitHealth(float NewVal);
 *
 * To use this in your game you can define something like this, and then add game-specific functions as necessary:
 * 
 *	#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
 *	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
 *	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
 *	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
 *	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
 * 
 *	ATTRIBUTE_ACCESSORS(UMyHealthSet, Health)
 */
```

## 声明

#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName)
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName)
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName)
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName)
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

在头文件中进行声明，通过此声明，我们可以用这任意一个宏，每个宏都对应着一个函数可以不用定义而去直接使用，在这里我们直接使用第一个宏对属性进行声明,相当于使用了四个宏

```
ATTRIBUTE_ACCESSORS(UAuraAttributeSet, MaxHealth);
```

## 使用

在构造函数中使用提前定义好的函数就可以了；

```
InitMaxHealth(100);
```
