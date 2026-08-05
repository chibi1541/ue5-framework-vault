---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection.GetSubsystemArrayCopy|FObjectSubsystemCollection::GetSubsystemArrayCopy]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.FindAndPopulateSubsystemArray|FSubsystemCollectionBase::FindAndPopulateSubsystemArray]]"
tags:
  - SubsystemCollection_h
---

```cpp
/** Get a list of Subsystems by type */
template<typename SubsystemType>
TArray<SubsystemType*> GetSubsystemArrayCopy(UClass* SubsystemClass) const
{
	if (const FSubsystemArray* Referenced = FindAndPopulateSubsystemArray(SubsystemClass))
	{
		return TArray<SubsystemType*>(reinterpret_cast<SubsystemType* const*>(Referenced->Subsystems.GetData()), Referenced->Subsystems.Num());
	}
	return {};
}
```

## 설명
- [[FObjectSubsystemCollection/FObjectSubsystemCollection.GetSubsystemArrayCopy|FObjectSubsystemCollection::GetSubsystemArrayCopy]]에서 타입 검사를 거친 뒤 실제로 호출되는 구현.
- [[FSubsystemCollectionBase/FSubsystemCollectionBase.FindAndPopulateSubsystemArray|FindAndPopulateSubsystemArray]]로 캐시된 서브시스템 목록을 얻고, 그 raw 데이터를 `SubsystemType*` 배열로 `reinterpret_cast`해서 **복사본**(Copy)을 만들어 반환한다.
- 이름 그대로 복사본이므로, 반환받은 배열을 순회하는 동안 컬렉션이 바뀌어도 안전하다. 대신 매 호출마다 배열 복사 비용이 든다.
- 캐시가 비어 있으면(`nullptr`) 빈 배열 `{}`을 반환한다.
