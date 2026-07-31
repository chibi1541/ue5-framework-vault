---
related:
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel.RemoveTickFunction|FTickTaskLevel::RemoveTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel.HasTickFunction|FTickTaskLevel::HasTickFunction]]"
  - "[[FTickFunction/FTickFunction.SetTickFunctionEnable|FTickFunction::SetTickFunctionEnable]]"
  - "[[FTickTaskManager/FTickTaskManager.AddTickFunction|FTickTaskManager::AddTickFunction]]"
tags:
  - TickTaskManager_cpp
---

```cpp
/** Add the tick function to the primary list **/
void AddTickFunction(FTickFunction* TickFunction)
{
    // haker: absolutely, there should NOT be in any queues of FTickTaskLevel
    check(!HasTickFunction(TickFunction));
    if (TickFunction->TickState == FTickFunction::ETickState::Enabled)
    {
        AllEnabledTickFunctions.Add(TickFunction);
        // haker: NewlySpawnedTickFunctions handles addition operations, but now skip it
        // - we'll cover when we look into the detail of how FTickFunction ticks
        if (bTickNewlySpawned)
        {
            NewlySpawnedTickFunctions.Add(TickFunction);
        }
    }
    else
    {
        // 혹시나 쿨링다운일 문제 상황을 체크하는 건가?
        check(TickFunction->TickState == FTickFunction::ETickState::Disabled);
        AllDisabledTickFunctions.Add(TickFunction);
    }
}
```

## 설명
- [[FTickTaskLevel/FTickTaskLevel.RemoveTickFunction|RemoveTickFunction()]]에서 경우의 수에 따른 예외 처리를 이미 끝냈기 때문에, 추가하는 쪽은 단순하다.
- 진입 시 [[FTickTaskLevel/FTickTaskLevel.HasTickFunction|HasTickFunction()]]으로 어떤 큐에도 들어 있지 않음을 보장한다.
- `TickState`가 `Enabled`면 `AllEnabledTickFunctions`에, 그 외에는 `AllDisabledTickFunctions`에 넣는다. 이때 `check`로 `Disabled`임을 확인하므로, `CoolingDown` 상태로 add 되는 경우는 문제 상황으로 걸러진다.
- tick phase 중(`bTickNewlySpawned`)이라면 `NewlySpawnedTickFunctions`에도 함께 넣는다.
