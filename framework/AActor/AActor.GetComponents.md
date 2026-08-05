---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[AActor/AActor.ForEachComponent_Internal|AActor::ForEachComponent_Internal]]"
  - "[[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents]]"
tags:
  - Actor_h
---

```cpp
/** 
 * get a direct reference to the components set rather than a copy with the null pointers removed 
 * WARNING: anything that could cause the component to change ownership or be destroyed will invalidate
 * this array, so use caution when iterating this set!
 */
const TSet<UActorComponent*>& GetComponents() const
{
    // haker: ObjectPtrDecal means removing ObjectPtr:
    // - OwnedComponents is TSet<TObjectPtr<UActorComponent>>
    // - Output is TSet<UActorComponent*>
    // TObjectPtr 뚜따 하고 싶은데 사용하는 함수
    return ObjectPtrDecay(OwnedComponents);
}
```

```cpp
/**
 * UActorComponent specialization of GetComponents() to avoid unnecessaray casts
 * it is recommended to use TArrays with a TInlineAllocator to potentially avoid memory allocation costs
 * TInlineComponentArray is defined to make this easier, for example:
 * {
 *  TInlineComponentArray<UActorComponent*> PrimComponents;
 *  Actor->GetComponents(PrimComponents);
 * } 
 */
// haker: you can specify whether you include ChildActors or not
template <class AllocatorType>
void GetComponents(TArray<UActorComponent*, AllocatorType>& OutComponents, bool bIncludeFromChildActors = false)
{
    OutComponents.Reset();
    ForEachComponent_Internal<UActorComponent>(UActorComponent::StaticClass(), bIncludeFromChildActor, [&](UActorComponent* InComp)
    {
        OutComponents.Add(InComp);
    });
}
```

## 설명
- 대표적인 `GetComponents` 함수는 두 가지 형태다.
- 첫 번째는 [[AActor/AActor|AActor]]의 `OwnedComponents`를 **복사 없이 그대로 참조**로 반환한다. `OwnedComponents`는 `TSet<TObjectPtr<UActorComponent>>`인데, `ObjectPtrDecay()`로 `TObjectPtr`를 벗겨내 `TSet<UActorComponent*>`로 만들어준다.
  - 참조이므로 순회 중에 컴포넌트의 소유권이 바뀌거나 파괴되면 그대로 무효화된다. 주석이 경고하는 부분.
- 두 번째는 `TArray`로 받는 템플릿 버전. `AllocatorType`을 지정할 수 있어 `TInlineComponentArray`(= `TInlineAllocator` 사용)로 받으면 힙 할당을 피할 수 있다. [[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents]]가 이 형태를 사용한다.
- 실제 순회는 [[AActor/AActor.ForEachComponent_Internal|ForEachComponent_Internal]]에 위임하며, `bIncludeFromChildActors`로 child actor의 컴포넌트까지 포함할지 선택할 수 있다.
