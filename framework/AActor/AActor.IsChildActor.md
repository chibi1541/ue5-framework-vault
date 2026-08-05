---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[SortActorsHierarchy|SortActorsHierarchy]]"
  - "[[AActor/AActor.ForEachComponent_Internal|AActor::ForEachComponent_Internal]]"
tags:
  - Actor_cpp
---

```cpp
bool AActor::IsChildActor() const
{
	return ParentComponent.IsValid();
}

// AActor's member variable
UPROPERTY()
TWeakObjectPtr<UChildActorComponent> ParentComponent;
```

## 설명
- 이 액터가 `UChildActorComponent`에 의해 생성/소유된 child actor인지 판별한다.
- 판정은 [[AActor/AActor|AActor]]의 멤버 `ParentComponent`(자신을 소유한 `UChildActorComponent`에 대한 weak ptr)의 유효성만 확인하면 된다.
- 액터와 액터는 계층 구조로 직접 묶을 수 없기 때문에, `UChildActorComponent`가 그 한계를 우회하는 장치다. `ParentComponent`는 그 역참조다.
