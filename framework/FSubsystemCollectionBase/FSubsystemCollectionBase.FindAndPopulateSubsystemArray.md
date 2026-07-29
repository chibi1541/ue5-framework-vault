---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.ForEachSubsystemOfClass|FSubsystemCollectionBase::ForEachSubsystemOfClass]]"
tags:
  - SubsystemCollection_cpp
---

```cpp
const FSubsystemCollectionBase::FSubsystemArray* FSubsystemCollectionBase::FindAndPopulateSubsystemArray(UClass* SubsystemClass) const
{
	// NOTE: There is no thread safety here for multiple threads trying to access a subsystem by a base class/interface.
	// Ideally there should be.
	const bool bIsInterface = SubsystemClass->IsChildOf<UInterface>();
	if (!SubsystemArrayMap.Contains(SubsystemClass))
	{
		FSubsystemArray NewList;
		for (auto Iter = SubsystemMap.CreateConstIterator(); Iter; ++Iter)
		{
			UClass* KeyClass = Iter.Key();
			if ((!bIsInterface && KeyClass->IsChildOf(SubsystemClass)) || 
				(bIsInterface && KeyClass->ImplementsInterface(SubsystemClass)))
			{
				NewList.Subsystems.Add(Iter.Value());
			}
		}
		if (NewList.Subsystems.Num())
		{
			return SubsystemArrayMap.Add(SubsystemClass, MakeUnique<FSubsystemArray>(MoveTemp(NewList))).Get();
		}
		// If nothing was found, we don't want to store this key in the map as we remove lists when they become empty.
		// We also don't pass the keys in SubsystemArrayMap to GC.
		return nullptr;
	}

	const TUniquePtr<FSubsystemArray>& List = SubsystemArrayMap.FindChecked(SubsystemClass);
	return List.Get();
}
```

## 설명
- [[FSubsystemCollectionBase/FSubsystemCollectionBase.ForEachSubsystemOfClass|FSubsystemCollectionBase::ForEachSubsystemOfClass]]에서 호출. `SubsystemArrayMap`에 캐시가 없으면 `SubsystemMap`을 순회해 `SubsystemClass`(또는 인터페이스)에 맞는 서브시스템 목록을 만들어 캐싱하고, 있으면 캐시를 그대로 반환한다.
- 스레드 안전성이 없다는 점을 소스 주석에서 스스로 지적하고 있다.
