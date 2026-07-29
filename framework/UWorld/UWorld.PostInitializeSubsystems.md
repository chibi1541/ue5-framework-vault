---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection.ForEachSubsystem|FObjectSubsystemCollection::ForEachSubsystem]]"
  - "[[UWorld/UWorld.InitWorld|UWorld::InitWorld]]"
tags:
  - World_cpp
---

```cpp
/** finalize initialization of all world subsystems */
void PostInitializeSubsystems()
{
		// 원래는 배열을 받아서 for문을 돌았는데 로직이 변경됨
		// 이것도 월드 내부의 subsystem을 관리하는 방식이 변경되어서 그런듯
		SubsystemCollection.ForEachSubsystem([](UWorldSubsystem* WorldSubsystem)
		{
				WorldSubsystem->PostInitialize();
				WorldSubsystem->EnsureHasCalledPostInitialize();
		});
}
```

## 설명
- [[UWorld/UWorld.InitWorld|UWorld::InitWorld]] 마지막 단계에서 호출. `SubsystemCollection`의 모든 서브시스템에 대해 [[FObjectSubsystemCollection/FObjectSubsystemCollection.ForEachSubsystem|FObjectSubsystemCollection::ForEachSubsystem]]으로 `PostInitialize()`를 호출해준다.
