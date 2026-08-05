---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[AActor/AActor.GetComponents|AActor::GetComponents]]"
  - "[[AActor/AActor.IsChildActor|AActor::IsChildActor]]"
tags:
  - Actor_h
---

```cpp
// haker: focus on how it uses ForEachComponent_Internal
// - bIsIncludeFromChildActor is dynamic variables (not static-compiled variable)
//   - so we call separate ForEachComponent_Internal with static-compiled variable
template <class ComponentType, typename Func>
void ForEachComponent_Internal(TSubclassOf<UActorComponent> ComponentClass, bool bIncludeFromChildActors, Func InFunc) const
{
		// 왜 ActorComponent인 경우를 나눴지?
    if (ComponentClass == UActorComponent::StaticClass())
    {
        if (bIncludeFromChildActors)
        {
		        // 템플릿 함수는 컴파일 타임에 인자 값이 확정되야 하기 때문에
		        // 아래와 같이 모든 버전을 일일히 정의함
            ForEachComponent_Internal<ComponentType, true /*bClassIsActorComponent*/, true /*bIncludeFromChildActors*/>(ComponentClass, InFunc);
        }
        else
        {
            ForEachComponent_Internal<ComponentType, true /*bClassIsActorComponent*/, false /*bIncludeFromChildActors*/>(ComponentClass, InFunc);
        }
    }
    else
    {
        if (bIncludeFromChildActors)
        {
            ForEachComponent_Internal<ComponentType, false /*bClassIsActorComponent*/, true /*bIncludeFromChildActors*/>(ComponentClass, InFunc);
        }
        else
        {
            ForEachComponent_Internal<ComponentType, false /*bClassIsActorComponent*/, false /*bIncludeFromChildActors*/>(ComponentClass, InFunc);
        }
    }
}
```

```cpp
/**
 * internal helper function to call a compile-time lambda on all components of a given-type
 * use template parameter bClassIsActorComponent to avoid doing unnecessary IsA checks when the ComponentClass is exactly UActorComponent
 * use template parameter bIncludeFromChildActors to recurse in to ChildActor components and find components of the appropriate type in those actors as well  
 */
template <class ComponentType, bool bClassIsActorComponent, bool bIncludeFromChildActors, typename Func>
void ForEachComponent_Internal(TSubclassOf<UActorComponent> ComponentClass, Func InFunc) const
{
    check(bClassIsActorComponent == false || ComponentClass == UActorComponent::StaticClass());
    check(ComponentClass->IsChildOf(ComponentType::StaticClass()));

		// 템플릿 함수가 컴파일 되는 타이밍에 인자 값이 정적으로 결정된 상태이므로
		// 템플릿 코드를 생성하는 과정에는 if문이 없어질 것임
    // static check, so that the most common case (bIncludeFromChildActors) doesn't need to allocate an additional array
    // haker: we are evaluating branch (condition) here:
    // - in compiled-time, it was fixed how the logic is flowed
    if (bIncludeFromChildActors)
    {
        TArray<AActor*, TInlineAllocator<NumInlinedActorComponents>> ChildActors;
        for (UActorComponent* OwnedComponent : OwnedComponents)
        {
            if (OwnedComponent)
            {
                // haker: by calling IsA(), we filter ActorComponent type
                if (bClassIsActorComponent || OwnedComponent->IsA(ComponentClass))
                {
                    InFunc(static_cast<ComponentType*>(OwnedComponent));
                }

                // haker: do you remember what ChildActorComponent is?
                // - for child-actor, we accumulate child-actor separately rather calling them in hierarchical manner
                if (UChildActorComponent* ChildActorComponent = Cast<UChildActorComponent>(OwnedComponent))
                {
                    if (AActor* ChildActor = ChildActorComponent->GetChildActor())
                    {
                        ChildActors.Add(ChildActor);
                    }
                }
            }
        }

        // haker: recursively call ForEachComponent_Internal() for each ChildActor
        for (AActor* ChildActor : ChildActors)
        {
            ChildActor->ForEachComponent_Internal<ComponentType, bClassIsActorComponent, bIncludeFromChildActors>(ComponentClass, InFunc);
        }
    }
    else
    {
        // haker: if we don't specify to include child-actor, just call OwnedComponents
        for (UActorComponent* OwnedComponent : OwnedComponents)
        {
            if (OwnedComponent)
            {
	            // 여기가 메인처리
	            // IsA 관계의 컴포넌트를 전부 찾아서 OutComponents로 반환
                if (bClassIsActorComponent || OwnedComponent->IsA(ComponentClass))
                {
                    InFunc(static_cast<ComponentType*>(OwnedComponent));
                }
            }
        }
    }
}
```

## 설명
- 오버로드가 두 개인 이유가 핵심이다. 앞의 것은 `bIncludeFromChildActors`를 **런타임 인자**로 받고, 뒤의 것은 그것을 **템플릿 인자(컴파일 타임 상수)** 로 받는다.
- 템플릿 인자는 컴파일 타임에 값이 확정되어야 하므로, 앞의 함수가 런타임 bool 조합(`bClassIsActorComponent` × `bIncludeFromChildActors`)을 4가지 명시적 호출로 풀어서 넘긴다. 그 결과 뒤쪽 구현의 `if (bIncludeFromChildActors)`는 코드 생성 시점에 한쪽만 남고 분기가 사라진다.
- `bClassIsActorComponent`를 따로 두는 이유는 `ComponentClass`가 정확히 `UActorComponent`일 때 불필요한 `IsA()` 검사를 생략하기 위해서다. 그래서 `bClassIsActorComponent || OwnedComponent->IsA(ComponentClass)` 형태로 short-circuit 된다.
- 실질적인 처리는 `OwnedComponents`를 돌면서 `IsA` 관계인 컴포넌트마다 `InFunc`(= [[AActor/AActor.GetComponents|GetComponents]]가 넘긴 람다)를 호출해 결과 배열에 담는 것이다.
- child actor를 포함하는 경로에서는 계층적으로 바로 재귀하지 않고, `UChildActorComponent`가 가리키는 `ChildActor`들을 `ChildActors` 배열에 먼저 모아둔 뒤 나중에 한 번에 재귀 호출한다. `TInlineAllocator`를 써서 이 임시 배열의 힙 할당도 피한다.
