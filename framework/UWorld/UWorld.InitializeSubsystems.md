---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[UWorld/UWorld.InitWorld|UWorld::InitWorld]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.Initialize|FSubsystemCollectionBase::Initialize]]"
tags:
  - World_cpp
---

```cpp
/** initialize all world subsystems */
void InitializeSubsystems()
{
    SubsystemCollection.Initialize(this);
}
```

## 설명
- [[UWorld/UWorld.InitWorld|UWorld::InitWorld]] 초반에 호출되어 `SubsystemCollection`(`FWorldSubsystemCollection`)을 초기화한다.
- 실제 초기화 로직은 [[FSubsystemCollectionBase/FSubsystemCollectionBase.Initialize|FSubsystemCollectionBase::Initialize]]로 위임된다.
