---
related:
  - "[[FTickTaskManager/FTickTaskManager.TickTaskLevelForLevel|FTickTaskManager::TickTaskLevelForLevel]]"
  - "[[FTickTaskLevel/FTickTaskLevel.AddTickFunction|FTickTaskLevel::AddTickFunction]]"
  - "[[FTickFunction/FTickFunction.RegisterTickFunction|FTickFunction::RegisterTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
tags:
  - TickTaskManager_cpp
---

```cpp
void AddTickFunction(ULevel* InLevel, FTickFunction* TickFunction)
{
    FTickTaskLevel* Level = TickTaskLevelForLevel(InLevel);
    // 아 여기서 Level에 AddTickFunction을 호출함
    Level->AddTickFunction(TickFunction);
    TickFunction->InternelData->TickTaskLevel = Level;
}
```

## 설명
- `ULevel`을 받아 그에 대응하는 [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]을 [[FTickTaskManager/FTickTaskManager.TickTaskLevelForLevel|TickTaskLevelForLevel()]]로 얻은 뒤, 실제 삽입은 [[FTickTaskLevel/FTickTaskLevel.AddTickFunction|FTickTaskLevel::AddTickFunction()]]에 위임한다.
- 삽입 후 `TickFunction->InternalData->TickTaskLevel`에 back pointer를 채워 넣는다. 이후 `SetTickFunctionEnable()` 등에서 자신이 속한 `FTickTaskLevel`을 이 포인터로 찾는다.
