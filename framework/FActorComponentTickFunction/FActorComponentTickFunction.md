---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FActorTickFunction/FActorTickFunction|FActorTickFunction]]"
  - "[[UActorComponent/UActorComponent.SetComponentTickEnabled|UActorComponent::SetComponentTickEnabled]]"
tags:
  - EngineBaseTypes_h
---

```cpp
UPROPERTY(EditDefaultsOnly, Category="ComponentTick")
struct FActorComponentTickFunction PrimaryComponentTick;class UActorComponent : public UObject, public IInterface_AssetUserData, public IAsyncPhysicsStateProcessor
{
#if UE_WITH_REMOTE_OBJECT_HANDLE
	TObjectPtr<class UActorComponent> Target;
#else
	/** Actor component that is the target of this tick */
	class UActorComponent*	Target;
#endif
}

// UActorComponent의 FTickFunction
/** Main tick function for the Component */
UPROPERTY(EditDefaultsOnly, Category="ComponentTick")
struct FActorComponentTickFunction PrimaryComponentTick;
```

## 설명
- [[FTickFunction/FTickFunction|FTickFunction]]을 상속한 `UActorComponent` 전용 tick function이다.
- `Target`이 tick 대상인 `UActorComponent`를 가리킨다. [[FActorTickFunction/FActorTickFunction|FActorTickFunction]]과의 차이는 이 `Target`이 `AActor`인지 `UActorComponent`인지 뿐이다.
- `UActorComponent`는 이를 `PrimaryComponentTick` 멤버로 갖고, [[UActorComponent/UActorComponent.SetComponentTickEnabled|UActorComponent::SetComponentTickEnabled]]가 이 객체를 통해 tick을 제어한다.
