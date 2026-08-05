---
related:
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection|FObjectSubsystemCollection]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.GetSubsystemArrayCopy|FSubsystemCollectionBase::GetSubsystemArrayCopy]]"
  - "[[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]"
tags:
  - SubsystemCollection_h
---

```cpp
/** Get a list of Subsystems by type */
template <typename TSubsystemClass>
TArray<TSubsystemClass*> GetSubsystemArrayCopy(const TSubclassOf<TSubsystemClass>& SubsystemClass) const
{
	// Force a compile time check that TSubsystemClass derives from TBaseType, the internal code only enforces it's a USubsystem
	TSubclassOf<TBaseType> SubsystemBaseClass = SubsystemClass;
	return FSubsystemCollectionBase::GetSubsystemArrayCopy<TSubsystemClass>(SubsystemBaseClass);
}
```

## 설명
- [[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]에서 `SubsystemCollection.GetSubsystemArray<UWorldSubsystem>(...)` 형태로 호출되는 진입점.
- 이 함수 자체가 하는 일은 **컴파일 타임 타입 검사뿐**이다. `TSubclassOf<TBaseType> SubsystemBaseClass = SubsystemClass;` 대입이 `TSubsystemClass`가 `TBaseType`(여기서는 `UWorldSubsystem`)의 파생 타입인지 컴파일 타임에 강제한다.
- base 쪽 구현은 `USubsystem`인지만 확인하므로, 이런 얇은 래퍼로 타입 안전성을 한 단계 더 얹는 구조다. 실제 조회는 [[FSubsystemCollectionBase/FSubsystemCollectionBase.GetSubsystemArrayCopy|FSubsystemCollectionBase::GetSubsystemArrayCopy]]가 수행한다.
