---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickFunction/FTickFunction.IsTickFunctionRegistered|FTickFunction::IsTickFunctionRegistered]]"
  - "[[FTickTaskManager/FTickTaskManager.AddTickFunction|FTickTaskManager::AddTickFunction]]"
  - "[[UActorComponent/UActorComponent.SetupActorComponentTickFunction|UActorComponent::SetupActorComponentTickFunction]]"
  - "[[ULevel/ULevel|ULevel]]"
tags:
  - TickTaskManager_cpp
---

```cpp
void RegisterTickFunction(ULevel* Level)
{
    if (!IsTickFunctionRegistered())
    {
        // only allow registration of tick if we are allowed on dedicated server, or we are not a dedicated server
        const UWorld* World = Level ? Level->GetWorld() : nullptr;

        // haker: we didn't care about dedicated server for now
        if (bAllowTickOnDedicatedServer || !(World && World->IsNetMode(NM_DedicatedServer)))
        {
            if (InternelData == nullptr)
            {
                InternalData.Reset(new FInternalData());
            }
            // haker: where FTickTaskLevel is allocated in ULevel?
            // - in ULevel's constructor, TickTaskLevel is allocated

            // haker: registering TickFuction means adding TickFunction
            // 아... 여기서 드디어 TickTaskManager에 FTickFunction을 등록
            FTickTaskManager::Get().AddTickFunction(Level, this);

            // haker: mark bRegistered as true
            // 등록 완료
            InternelData->bRegistered = true;
        }
    }
    else
    {
        // haker: ?! 
        check(FTickTaskManager::Get().HasTickFunction(Level, this));
    }
}
```

## 설명
- tick function을 [[ULevel/ULevel|ULevel]]에 등록하는 함수. 등록한다는 것은 결국 그 level의 `FTickTaskLevel` 큐에 자신을 add 하는 것이다.
- `bAllowTickOnDedicatedServer`가 false인데 dedicated server라면 등록하지 않는다.
- 여기서 처음으로 `InternalData`(`FInternalData`)가 할당된다. 등록되지 않은 tick function은 이 데이터를 갖지 않는 hot/cold 분리 설계이기 때문이다.
- [[FTickTaskManager/FTickTaskManager.AddTickFunction|FTickTaskManager::Get().AddTickFunction(Level, this)]]로 실제 등록을 수행하고, 마지막에 `InternalData->bRegistered = true`로 등록을 확정한다. 이 값이 [[FTickFunction/FTickFunction.IsTickFunctionRegistered|IsTickFunctionRegistered()]]의 판별 근거가 된다.
- 이미 등록된 상태로 다시 들어오면 `check`로 해당 level에 실제로 존재하는지만 검증한다.
