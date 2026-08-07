---
related:
  - "[[FTickTaskManager/FTickTaskManager.EndFrame|FTickTaskManager::EndFrame]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]"
tags:
  - TickTaskManager_cpp
---

```cpp
/** end a tick frame */
void EndFrame()
{
		// 마지막에 RescheduleList, CoolingDownList를 업데이트
    ScheduleTickFunctionCooldowns();

    // haker: note that TickNewlySpawned is set as false
    bTickNewlySpawned = false;
}
```

## 설명
- [[FTickTaskManager/FTickTaskManager.EndFrame|FTickTaskManager::EndFrame()]]에서 레벨마다 호출된다.
- `ScheduleTickFunctionCooldowns()`로 `TickFunctionsToReschedule`(reschedule 대기 목록)을 [[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|AllCoolingDownTickFunctions]]에 반영한다. 즉 cooldown 리스트 갱신은 프레임 끝에서 한 번에 이뤄진다.
- `bTickNewlySpawned`를 false로 되돌려 tick phase 종료를 표시한다.
