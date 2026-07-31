---
related:
  - "[[UActorComponent/UActorComponent.RegisterComponentWithWorld|UActorComponent::RegisterComponentWithWorld]]"
  - "[[UActorComponent/UActorComponent.RegisterComponentTickFunctions|UActorComponent::RegisterComponentTickFunctions]]"
  - "[[UActorComponent/UActorComponent.OnRegister|UActorComponent::OnRegister]]"
  - "[[AActor/AActor.HandleRegisterComponentWithWorld|AActor::HandleRegisterComponentWithWorld]]"
tags:
  - ActorComponent_cpp
---

```cpp
// bRegister = true
void RegisterAllComponentTickFunctions(bool bRegister)
{
    // components don't have tick functions until they are registered with the world
    // haker: bRegistered becomes true when OnRegister() is called (== registered with the world)
    // OnRegister가 호출되면 true가 됨
    // 그 말은 다른 월드에 state가 만들어진 후에야 TickFunction 등록이 된다는 의미
    if (bRegistered)
    {
        // prevent repeated redundant attempts
        if (bTickFunctionsRegistered != bRegister)
        {
            // 얘를 오버라이드 하는게 가능
            RegisterComponentTickFunctions(bRegister);
            bTickFunctionsRegistered = bRegister;
        }
        
        if (bAsyncPhysicsTickEnabled)
        {
            // haker: skip AsyncPhysicsTickEnabled():
            // - this is the TickFunction as we covered that multiple tick functions can be registered
            // 이건 단순히 TickFunction이 하나의 Actor, Component에 복수개 있을 수 있다 정도만 인식
            RegisterAsyncPhysicsTickEnabled(bRegister);
        }
    }
}
```

## 설명
- 컴포넌트가 world에 register 되기 전까지는 tick function을 갖지 않는다. `bRegistered`는 [[UActorComponent/UActorComponent.OnRegister|OnRegister()]]가 호출될 때 true가 되므로, render/physics world에 state가 만들어진 뒤에야 tick function 등록이 진행된다는 의미다.
- `bTickFunctionsRegistered != bRegister` 조건으로 같은 등록/해제를 중복 수행하지 않게 막는다.
- 실제 등록은 오버라이딩 가능한 [[UActorComponent/UActorComponent.RegisterComponentTickFunctions|RegisterComponentTickFunctions()]]에 위임한다.
- `bAsyncPhysicsTickEnabled`면 `RegisterAsyncPhysicsTickEnabled()`도 호출한다. 하나의 Actor/Component가 복수의 tick function을 가질 수 있다는 정도로 이해하고 넘어간다.
