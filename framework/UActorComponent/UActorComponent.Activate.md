---
related:
  - "[[UActorComponent/UActorComponent.OnRegister|UActorComponent::OnRegister]]"
  - "[[UActorComponent/UActorComponent.SetComponentTickEnabled|UActorComponent::SetComponentTickEnabled]]"
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
tags:
  - ActorComponent_cpp
---

```cpp
/** activates the SceneComponent, should be overriden by native child classes */
virtual void Activate(bool bReset=false)
{
    // haker: before we are getting into the details of Activate, see FTickFunction related classes first:
    if (bReset || ShouldActivate() == true)
    {
        SetComponentTickEnabled(true);
        
        SetActiveFlag(true);

        OnComponentActivated.Broadcast(this, bReset);
    }
}
```

## 설명
- 컴포넌트를 활성화한다. native 자식 클래스에서 오버라이딩하는 것을 전제로 한 가상 함수다.
- `bReset`이 true거나 `ShouldActivate()`가 true일 때만 동작한다.
- 활성화의 실체는 세 가지: [[UActorComponent/UActorComponent.SetComponentTickEnabled|UActorComponent::SetComponentTickEnabled]]로 tick을 켜고, `SetActiveFlag(true)`로 active 플래그를 세우고, `OnComponentActivated` 델리게이트를 broadcast 한다.
- tick을 켠다는 것이 곧 [[FTickFunction/FTickFunction|FTickFunction]]의 상태를 바꾸는 것이므로, 자세한 동작은 tick function 관련 클래스들을 먼저 봐야 한다.
