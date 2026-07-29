---
related:
  - "[[USubsystem/USubsystem|USubsystem]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.ForEachSubsystemOfClass|FSubsystemCollectionBase::ForEachSubsystemOfClass]]"
  - "[[UWorld/UWorld.PostInitializeSubsystems|UWorld::PostInitializeSubsystems]]"
tags:
  - SubsystemCollection_h
---

```cpp
/** Perform an operation on all subsystems in the collection */
void ForEachSubsystem(TFunctionRef<void(TBaseType*)> Operation, const TSubclassOf<TBaseType>& SubsystemClass = {}) const
{
	ForEachSubsystemOfClass(SubsystemClass, [Operation=MoveTemp(Operation)](USubsystem* Subsystem){
		Operation(CastChecked<TBaseType>(Subsystem));
	});
}
```

## 설명
- [[UWorld/UWorld.PostInitializeSubsystems|UWorld::PostInitializeSubsystems]]에서 `SubsystemCollection.ForEachSubsystem(...)`으로 호출된다.
- 실제 순회는 [[FSubsystemCollectionBase/FSubsystemCollectionBase.ForEachSubsystemOfClass|FSubsystemCollectionBase::ForEachSubsystemOfClass]]에 위임하고, 콜백 안에서 `TBaseType*`(여기서는 `UWorldSubsystem*`)으로 캐스팅해준다.
