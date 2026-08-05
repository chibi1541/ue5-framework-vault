---
related:
  - "[[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents]]"
  - "[[USceneComponent/USceneComponent|USceneComponent]]"
  - "[[AActor/AActor|AActor]]"
tags:
  - Actor_cpp
---

```cpp
/**
 * walks through components hierarchy and returns closest to root parent component that is unregistered
 * only for components that belong to the same owner 
 */
// haker: return closest un-registered parent component:
// - constraints:
//   - AttachParent should have same owner(AActor) to Component
//   - AttachParent should be bAutoRegister as true
static USceneComponent* GetUnregisteredParent(UActorComponent* Component)
{
    USceneComponent* ParentComponent = nullptr;
    USceneComponent* SceneComponent = Cast<USceneComponent>(Component);

    while (SceneComponent
        // haker: we check all conditions for Component's AttachParent
        // - whether AttachParent's owner and Component's owner is same
        // - whether Component's AttachParent is registered
        //   --> this is final condition we exactly look for
        && SceneComponent->GetAttachParent()
        && SceneComponent->GetAttachParent()->GetOwner() == Component->GetOwner()
        && !SceneComponent->GetAttachParent()->IsRegistered())
    {
        SceneComponent = SceneComponent->GetAttachParent();
        if (SceneComponent->bAutoRegister && IsValidChecked(SceneComponent))
        {
            // we found unregistered parent that should be registered
            // but keep looking up the tree
            ParentComponent = SceneComponent;
        }
    }

    return ParentComponent;
}
```

## 설명
- 인자로 받은 컴포넌트와 **같은 Owner(`AActor`)** 를 가지면서 `bAutoRegister`가 true인데도 아직 register되지 않은, **가장 최상위(root에 가까운)** 부모 컴포넌트를 찾아 반환한다.
- [[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents]]에서 호출되며, 부모가 먼저 등록되어야 자식이 `GetLocation()` 같은 호출을 안전하게 할 수 있기 때문에 필요하다.
- `AttachParent`를 타고 올라가는 조건은 세 가지: (1) `AttachParent`가 존재할 것, (2) `AttachParent`의 owner가 `Component`의 owner와 같을 것(다른 액터로 넘어가지 않게), (3) `AttachParent`가 아직 unregistered일 것.
- 루프를 돌면서 조건을 만족하는 것을 발견해도 바로 반환하지 않고 `ParentComponent`에 계속 덮어쓴다. 즉 **가장 마지막으로 발견된(= 가장 위쪽) 미등록 부모**가 반환값이 된다.
- 모든 부모가 이미 등록되어 있으면 `nullptr`를 반환한다.
- 인자는 `UActorComponent*`지만 계층 구조는 [[USceneComponent/USceneComponent|USceneComponent]]에만 존재하므로 먼저 `Cast`로 걸러낸다. `USceneComponent`가 아니면 루프가 아예 돌지 않고 `nullptr`가 나온다.
