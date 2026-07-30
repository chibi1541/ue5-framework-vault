---
related:
  - "[[UActorComponent/UActorComponent.ExecuteRegisterEvents|UActorComponent::ExecuteRegisterEvents]]"
  - "[[UActorComponent/UActorComponent.Activate|UActorComponent::Activate]]"
  - "[[USceneComponent/USceneComponent|USceneComponent]]"
  - "[[AActor/AActor|AActor]]"
tags:
  - ActorComponent_cpp
---

```cpp
/** called when a component is registered, after Scene is set, but before CreateRenderState_Concurrent or OnCreatePhysicsState are called */
virtual void OnRegister()
{
    bRegistered = true;

    // haker: 
    // as the comment describes, this function is called before Create[Render|Phyiscs]State:
    // - it infers we should update all component transforms correctly
    // - if a component is USceneComponent, it should update all children's transforms
    UpdateComponentToWorld();

		// Register 단계에서 Activate를 수행
    // haker: when bAutoActivate is enabled, it try to activate on registeration stage
    if (bAutoActivate)
    {
        AActor* Owner = GetOwner();

        // haker: Owner == nullptr which is exact condition we matched for LineBatcher
        if (!WorldPrivate->IsGameWorld() 
            // haker: Owner can be nullptr, if a component is contained directly by UWorld
            // e.g. LineBatcher in UWorld
            || Owner == nullptr 
            || Owner->IsActorInitialized)
        {
            // haker: as we covered before, Activate() is usually called on BeginPlay(
            Activate(true);
        }
    }
}
```

## 설명
- `Scene`이 설정된 뒤, `CreateRenderState_Concurrent`/`OnCreatePhysicsState`보다 **먼저** 호출된다. 즉 render/physics state를 만들기 전에 transform이 올바르게 갱신되어 있어야 한다.
- `UpdateComponentToWorld()`로 컴포넌트 transform을 갱신한다. [[USceneComponent/USceneComponent|USceneComponent]]라면 자식들의 transform까지 갱신해야 한다.
- `bAutoActivate`가 켜져 있으면 register 단계에서 곧바로 [[UActorComponent/UActorComponent.Activate|UActorComponent::Activate]]를 호출한다. 원래 `Activate()`는 보통 `BeginPlay()`에서 호출된다.
- activate 조건은 game world가 아니거나, owner가 nullptr이거나, owner가 이미 초기화된 경우다. owner가 nullptr인 경우는 `UWorld`가 직접 들고 있는 LineBatcher 같은 컴포넌트에 해당한다.
