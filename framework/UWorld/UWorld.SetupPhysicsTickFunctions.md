---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[FPhysicsTickFunction/FPhysicsTickFunction|FPhysicsTickFunction]]"
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickFunction/FTickFunction.IsTickFunctionRegistered|FTickFunction::IsTickFunctionRegistered]]"
  - "[[FTickFunction/FTickFunction.RegisterTickFunction|FTickFunction::RegisterTickFunction]]"
  - "[[FTickFunction/FTickFunction.AddPrerequisite|FTickFunction::AddPrerequisite]]"
  - "[[Enum/ETickingGroup|ETickingGroup]]"
  - "[[ULevel/ULevel|ULevel]]"
tags:
  - PhysLevel_cpp
---

```cpp
/** set up the physics tick function if they aren't already */
void SetupPhysicsTickFunctions(float DeltaSeconds)
{
    StartPhysicsTickFunction.bCanEverTick = true;
    StartPhysicsTickFunction.Target = this;

    EndPhysicsTickFunction.bCanEverTick = true;
    EndPhysicsTickFunction.Target = this;

    bool bEnablePhysics = bShouldSimulatePhysics;

    // see if we need to update tick registration
    // haker: if both tick functions are NOT registered yet, register both functions
    // - note that we can define/register TickFunction in any class like this
    //   - no need to be in UObject/AActor/UActorComponent
    bool bNeedToUpdateTickRegistration = (bEnablePhysics != StartPhysicsTickFunction.IsTickFunctionRegistered())
        || (bEnablePhysics != EndPhysicsTickFunction.IsTickFunctionRegistered());

    // haker: note that if bEnablePhysics == false, we are going to unregister both tick functions
    if (bNeedToUpdateTickRegistration && PersistentLevel)
    {
        // IsTickFunctionRegistered => FInternalData 내부의 bRegistered 플래그를 체크
        if (bEnablePhysics && !StartPhysicsTickFunction.IsTickFunctionRegistered())
        {
            StartPhysicsTickFunction.TickGroup = TG_StartPhysics;
            StartPhysicsTickFunction.RegisterTickFunction(PersistentLevel);
        }
        else if (!bEnablePhysics && StartPhysicsTickFunction.IsTickFunctionRegistered())
        {
            StartPhysicsTickFunction.UnRegisterTickFunction();
        }

        if (bEnablePhysics && !EndPhysicsTickFunction.IsTickingFunctionRegistered())
        {
            EndPhysicsTickFunction.TickGroup = TG_EndPhysics;
            EndPhysicsTickFunction.RegisterTickFunction(PersistentLevel);
            // Tick 간의 순서 설정
            // AddPrerequisite로 선제 조건을 넣는 것으로 그 다음에 Tick이 돌도록 설정 가능
            // haker: EndPhysicsTickFunction has prerequisite to StartPhysicsTickFunction:
            // - it means that EndPhysicsTickFunction can start only if StartPhysicsTickFunction is finished
            EndPhysicsTickFunction.AddPrerequisite(this, StartPhysicsTickFunction);
        }
        else if (!bEnablePhysics && EndPhysicsTickFunction.IsTickFunctionRegistered())
        {
            EndPhysicsTickFunction.RemovePrerequisite(this, StartPhysicsTickFunction);
            EndPhysicsTickFunction.UnRegisterTickFunction();
        }  
    }
}
```

## 설명
- [[UWorld/UWorld|UWorld]]가 멤버로 들고 있는 `StartPhysicsTickFunction` / `EndPhysicsTickFunction`을 초기화하고 `PersistentLevel`에 등록/해제하는 함수다.
- tick function은 `UObject`/`AActor`/`UActorComponent`가 아니어도 아무 클래스에서나 선언하고 등록할 수 있다는 것을 보여주는 예다. 여기서는 `UWorld`가 직접 들고 있다.
- 등록 갱신이 필요한지는 `bEnablePhysics`와 [[FTickFunction/FTickFunction.IsTickFunctionRegistered|IsTickFunctionRegistered()]] 결과가 어긋나는지로 판단한다. `bEnablePhysics == false`면 두 tick function 모두 unregister 한다.
- `StartPhysicsTickFunction`은 `TG_StartPhysics`, `EndPhysicsTickFunction`은 `TG_EndPhysics` group으로 등록된다([[Enum/ETickingGroup|ETickingGroup]]).
- 핵심은 [[FTickFunction/FTickFunction.AddPrerequisite|AddPrerequisite()]]로 tick function 사이의 선후 관계를 세우는 부분이다. `EndPhysicsTickFunction`의 prerequisite로 `StartPhysicsTickFunction`을 걸어, start가 끝나야 end가 시작될 수 있게 만든다.
- unregister 경로에서는 `RemovePrerequisite()`로 의존 관계를 먼저 끊은 뒤 tick function을 해제한다.
- 나머지 흐름은 지금까지 본 [[FTickFunction/FTickFunction.RegisterTickFunction|RegisterTickFunction()]] / `UnRegisterTickFunction()` 패턴의 반복이다.
