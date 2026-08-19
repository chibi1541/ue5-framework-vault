---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickPrerequisite/FTickPrerequisite|FTickPrerequisite]]"
  - "[[FTickFunction/FTickFunction.IsTickFunctionRegistered|FTickFunction::IsTickFunctionRegistered]]"
  - "[[UWorld/UWorld.SetupPhysicsTickFunctions|UWorld::SetupPhysicsTickFunctions]]"
tags:
  - TickTaskManager_cpp
---

```cpp
// TargetObject => UWorld, TargetTickFunction=>StartPhysicsTickFunction
/** adds a tick function to the list of prerequisites... in other words, add the requirement that TargetTickFunction is called before this tick function */
void AddPrerequisite(UObject* TargetObject, struct FTickFunction& TargetTickFunction)
{
    // haker: CanTick is determined by bRegistered && bCanEverTick
    const bool bThisCanTick = (bCanEverTick || IsTickFunctionRegistered());
    const bool bTargetCanTick = (TargetTickFunction.bCanEverTick || TargetTickFunction.IsTickFunctionRegistered());
    if (bThisCanTick && bTargetCanTick)
    {
        // haker: now we can understand what Prerequisites is
        // - the detail of how it works will be seen in the code later
        Prerequisites.AddUnique(FTickPrerequisite(TargetObject, TargetTickFunction));
    }
}
```

## 설명
- `TargetTickFunction`이 자신보다 **먼저** 호출되어야 한다는 선후 조건을 추가한다.
- 양쪽 tick function이 모두 tick 가능한 상태일 때만 등록한다. tick 가능 여부는 `bCanEverTick`이거나 [[FTickFunction/FTickFunction.IsTickFunctionRegistered|IsTickFunctionRegistered()]]가 true인지로 판단한다.
- 실제 저장은 [[FTickPrerequisite/FTickPrerequisite|FTickPrerequisite]]를 만들어 `Prerequisites` 배열에 `AddUnique()`로 넣는 형태라 중복 등록은 자동으로 걸러진다.
- `TargetObject`는 대상 tick function을 소유한 `UObject`로, 이후 그 prerequisite가 아직 살아 있는지 판별하는 데 쓰인다.
- 대표적인 사용처가 [[UWorld/UWorld.SetupPhysicsTickFunctions|UWorld::SetupPhysicsTickFunctions()]]이며, `EndPhysicsTickFunction`에 `StartPhysicsTickFunction`을 prerequisite로 건다.
