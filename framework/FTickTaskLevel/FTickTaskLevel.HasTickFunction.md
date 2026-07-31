---
related:
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]"
  - "[[FTickTaskLevel/FTickTaskLevel.AddTickFunction|FTickTaskLevel::AddTickFunction]]"
tags:
  - TickTaskManager_cpp
---

```cpp
bool HasTickFunction(FTickFunction* TickFunction)
{
	return AllEnabledTickFunctions.Contains(TickFunction) || AllDisabledTickFunctions.Contains(TickFunction) || AllCoolingDownTickFunctions.Contains(TickFunction);
}
```

## 설명
- tick function이 이 [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]의 큐 중 하나라도 들어 있는지 확인한다. [[FTickTaskLevel/FTickTaskLevel.AddTickFunction|AddTickFunction()]]의 `check`에 쓰인다.
- `AllEnabledTickFunctions`/`AllDisabledTickFunctions`는 `TSet`이라 `Contains()`가 해시 조회지만, `AllCoolingDownTickFunctions`는 linked list다. [[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]의 `Contains()`는 `Head`부터 `Next`를 따라가며 순회하는 전형적인 linked list 탐색으로 구현되어 있다.
