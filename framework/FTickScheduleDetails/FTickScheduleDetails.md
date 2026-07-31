---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]"
  - "[[FTickTaskLevel/FTickTaskLevel.RemoveTickFunction|FTickTaskLevel::RemoveTickFunction]]"
tags:
  - TickTaskManager_cpp
---

```cpp
struct FTickScheduleDetails
{
    FTickFunction* TickFunction;
    // 쿨타임에 대한 정보
    float Cooldown;
    bool bDeferredRemove;
};
```

## 설명
- 각 [[FTickFunction/FTickFunction|FTickFunction]]이 어떻게 스케줄될지를 기술하는 별도 데이터. [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]의 `TickFunctionsToReschedule` 배열 원소로 쓰인다.
- 별도 구조체로 분리한 이유: cooldown을 관리하는 매니저가 [[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]](linked list)를 순회하며 cooldown을 체크하면, 메모리상 흩어져 있는 `AActor`/`UActorComponent`의 tick function을 따라가야 해서 cache hit율이 떨어진다.
- 그래서 스케줄링에 필요한 정보(`Cooldown`, `bDeferredRemove`)만 모아 연속된 배열로 관리한다.
