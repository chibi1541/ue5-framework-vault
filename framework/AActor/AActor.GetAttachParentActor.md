---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[USceneComponent/USceneComponent|USceneComponent]]"
  - "[[SortActorsHierarchy|SortActorsHierarchy]]"
tags:
  - Actor_cpp
---

```cpp
AActor* AActor::GetAttachParentActor() const
{
	if (GetRootComponent() && GetRootComponent()->GetAttachParent())
	{
		return GetRootComponent()->GetAttachParent()->GetOwner();
	}

	return nullptr;
}
```

## 설명
- 액터끼리는 직접적인 부모-자식 포인터를 갖지 않는다. 대신 자신의 `RootComponent`가 어떤 [[USceneComponent/USceneComponent|USceneComponent]]에 attach 되어 있는지를 보고, 그 컴포넌트의 owner 액터를 부모로 간주한다.
- 즉 `AActor` → `RootComponent` → `AttachParent` → `GetOwner()` 경로로 AActor-AActor 관계를 역추적한다.
- `RootComponent`가 없거나 attach 되지 않았다면 부모가 없으므로 `nullptr`을 반환한다.
- [[SortActorsHierarchy|SortActorsHierarchy()]]에서 액터의 depth를 계산할 때 이 함수를 사용한다.
