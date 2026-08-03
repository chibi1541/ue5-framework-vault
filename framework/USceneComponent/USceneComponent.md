---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[UActorComponent/UActorComponent.OnRegister|UActorComponent::OnRegister]]"
  - "[[SortActorsHierarchy|SortActorsHierarchy]]"
  - "[[AActor/AActor.GetAttachParentActor|AActor::GetAttachParentActor]]"
  - "[[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents]]"
tags:
  - SceneComponent_h
---

```cpp
/**
 * a SceneComponent has a transform and supports attachment, but has no rendering or collision capabilities
 * useful as a 'dummy' component in hierarchy to offset others 
 */
// scene-graph : 부모의 상대 좌표를 가지므로써 트랜스폼을 용이하게 하기 위한 구조
// haker: as we covered, USceneComponent supports scene-graph:
// - what is scene-graph for?
//   - supports hierarichy:
//     - representative example is 'transforms'
class USceneComponent : public UActorComponent
{
    /** get the SceneComponent we are attached to */
    USceneComponent* GetAttachParent() const
    {
        return AttachParent;
    }

    // haker: with AttachParent and AttachChildren, it supports tree-structure for scene-graph

    /** what we are currently attached to. if valid, RelativeLocation etc. are used relative to this object */
    UPROPERTY(ReplicatedUsing=OnRep_AttachParent)
    TObjectPtr<USceneComponent> AttachParent;

    /** list of child SceneComponents that are attached to us. */
    UPROPERTY(ReplicatedUsing = OnRep_AttachChildren, Transient)
    TArray<TObjectPtr<USceneComponent>> AttachChildren;
};
```

## 설명
- 단순히 좌표를 가지는 컴포넌트가 아니라, 자신의 부모와 자식 정보를 가지므로써 계층 구조(scene-graph)를 성립시키는 컴포넌트다.
- transform과 attachment를 지원하지만 렌더링/충돌 기능은 없어, 다른 컴포넌트들의 offset을 잡는 'dummy' 컴포넌트로도 유용하다.
- `AttachParent`와 `AttachChildren`으로 트리 구조를 만든다. `RelativeLocation` 등은 `AttachParent` 기준의 상대 좌표다.
- `AttachChildren`은 `Transient`라 직렬화되지 않는다. 즉 저장 시에는 자신의 부모 정보만 저장된다.
