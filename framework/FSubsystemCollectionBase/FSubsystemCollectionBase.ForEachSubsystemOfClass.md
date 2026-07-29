---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[USubsystem/USubsystem|USubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.FindAndPopulateSubsystemArray|FSubsystemCollectionBase::FindAndPopulateSubsystemArray]]"
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection.ForEachSubsystem|FObjectSubsystemCollection::ForEachSubsystem]]"
tags:
  - SubsystemCollection_cpp
---

```cpp
void FSubsystemCollectionBase::ForEachSubsystemOfClass(UClass* SubsystemClass, TFunctionRef<void(USubsystem*)> Operation) const
{
	if (SubsystemClass == nullptr)
	{
		SubsystemClass = USubsystem::StaticClass();
	}

	if (const FSubsystemArray* List = FindAndPopulateSubsystemArray(SubsystemClass))
	{
		// iteration으로 도는 동안에 안에 있는 값 지우지 않도록 하기 위한 세이프 가드?
		TGuardValue<bool> IterationGuard{List->bIsIterating, true};
		for (int32 i=0; i < List->Subsystems.Num(); ++i)
		{
			Operation(List->Subsystems[i]);
		}
	}
}
```

## 설명
- [[FObjectSubsystemCollection/FObjectSubsystemCollection.ForEachSubsystem|FObjectSubsystemCollection::ForEachSubsystem]]에서 호출. [[FSubsystemCollectionBase/FSubsystemCollectionBase.FindAndPopulateSubsystemArray|FSubsystemCollectionBase::FindAndPopulateSubsystemArray]]로 캐시된 `FSubsystemArray`를 얻어 순회하며 `Operation`을 실행한다.
- `IterationGuard`는 순회 도중 리스트가 변경되지 않도록 하는 안전장치로 추정.
