---
related:
  - "[[UActorComponent/UActorComponent.RegisterComponentWithWorld|UActorComponent::RegisterComponentWithWorld]]"
  - "[[UActorComponent/UActorComponent.OnRegister|UActorComponent::OnRegister]]"
tags:
  - ActorComponent_cpp
---

```cpp
/** calls OnRegister, CreateRenderState_Concurrent and OnCreatePhysicsState */
void ExecuteRegisterEvents(FRegisterComponentContext* Context = nullptr)
{
    if (!bRegistered)
    {
        OnRegister();
    }

    if (FApp::CanEverRender() && !bRenderStateCreated && WorldPrivate->Scene 
        // haker: look ShouldCreateRenderState() 
        && ShouldCreateRenderState())
    {
        // haker:
        // - remember this
        // - this is the place that we add primitive to render world (world's scene == FScene)
        CreateRenderState_Concurrent(Context);
    }

    // haker:
    // we are not going to do deep-dive on physics for a while.
    CreatePhysicsState(/*bAllowDeferral=*/true);

    // haker:
    // - create[render|physics]state means 'creating the reflection of main world's state into each render-world and physics world'
}
```

## 설명
- [[UActorComponent/UActorComponent.RegisterComponentWithWorld|UActorComponent::RegisterComponentWithWorld]] 내부에서 호출되며, register 과정의 실제 이벤트 3종을 순서대로 실행한다: `OnRegister()` → `CreateRenderState_Concurrent()` → `CreatePhysicsState()`.
- [[UActorComponent/UActorComponent.OnRegister|UActorComponent::OnRegister]]는 `bRegistered`가 false일 때만 호출된다.
- `CreateRenderState_Concurrent`는 primitive를 render world(`UWorld::Scene == FScene`)에 추가하는 지점이다. 렌더링이 가능하고(`FApp::CanEverRender()`), 아직 render state가 없고, `Scene`이 존재하며 `ShouldCreateRenderState()`가 true일 때만 수행된다.
- `Create[Render|Physics]State`의 의미는 main world의 상태를 render world / physics world 각각에 반영한 "분신"을 만드는 것이다.
