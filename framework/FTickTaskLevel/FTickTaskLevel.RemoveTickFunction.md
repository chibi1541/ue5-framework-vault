---
related:
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]"
  - "[[FTickScheduleDetails/FTickScheduleDetails|FTickScheduleDetails]]"
  - "[[FTickFunction/FTickFunction.SetTickFunctionEnable|FTickFunction::SetTickFunctionEnable]]"
  - "[[FTickTaskLevel/FTickTaskLevel.AddTickFunction|FTickTaskLevel::AddTickFunction]]"
tags:
  - TickTaskManager_cpp
---

```cpp
void RemoveTickFunction(FTickFunction* TickFunction)
{
    // haker: AllEnabledTickFunctions, AllDisabledTickFunctions, TickFunctionsToReschedule and AllCoolingDownTickFunctions(linked-list) are need to be synchronized
    switch(TickFunction->TickState)
    {
    case FTickFunction::ETickState::Enabled:
        // Enabled 상태이지만 직전 리스케쥴 상태였던 혹은 쿨링 다운 상태였던 것을 의미
        if (TickFunction->InternalData->bWasInternal)
        {
            // an enabled function with a tick interval could be in either the enabled or cooling down list
            // haker: 
            // - what is the meaning of AllEnabledTickFunctions.Remove() equals 0?
            //   - it is in cooling-down list or in reschedule list
            // Remove는 삭제를 실패 했을 때 0을 반환함
            if (AllEnabledTickFunctions.Remove(TickFunction) == 0)
            {
                // 아직 Enable 배열에 들어가 있지 않았던 상황
                auto FindTickFunctionInRescheduleList = [TickFunction](const FTickScheduleDetails& TSD)
                {
                    return (TSD.TickFunction == TickFunction);
                };
                // haker: note that TickFunctionsToReschedule is the array (liner-search)
                int32 Index = TickFunctionsToReschedule.IndexOfByPredicate(FindTickFunctionInRescheduleList);
                bool bFound = Index != INDEX_NONE;

                // haker: 
                // 1. remove TickFunction from TickFunctionsToReschedule
                if (bFound)
                {
                    // Threadsafe Array
                    // 빼는 처리는 너무 어려움;;
                    TickFunctionsToReschedule.RemoveAtSwap(Index);
                }

                // haker:
                // 2. remove TickFunction from AllCoolingDownTickFunctions
                // - if we already found the TickFunction in TickFunctionsToReschedule(using bFound), the TickFunction is not in cooling-down list
                FTickFunction* PrevComparisonFunction = nullptr;
                FTickFunction* ComparisonFunction = AllCoolingDownTickFunctions.Head;
                // 아래 조건으로 위에 리스케쥴에서 해당 FTickFunction을 찾았다면 스킵
                while (ComparisonFunction && !bFound)
                {
                    // 쿨링 다운은 링크드 리스트 형태로 관리되고 있으므로
                    // 아래와 같이 리스트에서 중간을 삭제하는 처리가 진행 중
                    if (ComparisonFunction == TickFunction)
                    {
                        bFound = true;
                        // 삭제하려는 FTickFunction이 헤드가 아닌 경우
                        if (PrevComparisonFunction)
                        {
                            PrevComparisonFunction->InternalData->Next = TickFunction->InternalData->Next;
                        }
                        // 헤드인 경우
                        else
                        {
                            // haker: it starts from AllCoolingDownTickFunctions.Head, so matches the below condition
                            check(TickFunction == AllCoolingDownTickFunctions.Head);
                            AllCoolingDownTickFunctions.Head = TickFunction->InternalData->Next;
                        }
                        TickFunction->InternalData->Next = nullptr;
                    }
                    else
                    {
                        // 삭제할 노드가 아니라면 탐색을 진행
                        PrevComparisonFunction = ComparisonFunction;
                        ComparisonFunction = ComparisonFunction->InternalData->Next;
                    }
                }
                // haker: when TickFunction::TickState == Enabled and it is not in AllEnabledTickFunctions, it should be found in TickFunctionsToReschedule or AllCoolingDownTickFunctions
                check(bFound);
            }
        }
        else
        {
            // haker:
            // it it is not internal, it already ticking so we can remove AllEnabledTickFunctions
            // what is verify? -> assert 같은 건가?
            // 아래 조건은 무조건 만족시켜야지 그렇지 않으면 문제 상황
            verify(AllEnabledTickFunctions.Remove(TickFunction) == 1);
        }
        break;

    case FTickFunction::ETickState::Disabled:
        // Disable 이면 묻지도 따지지도 말고 Diable 배열에 있을 것
        // haker: when TickState is Disabled, it should be in AllDisabledTickFunctions
        verify(AllDisabledTickFunctions.Remove(TickFunction) == 1);
        break;

    case FTickFunction::ETickState::CoolingDown:
        // 위에서 쿨링다운 리스트 뒤지던 것과 동일
        auto FindTickFunctionInRescheduleList = [TickFunction](const FTickScheduleDetails& TSD)
        {
            return (TSD.TickFunction == TickFunction);
        };
        int32 Index = TickFunctionsToReschedule.IndexOfByPredicate(FindTickFunctionInRescheduleList);
        bool bFound = Index != INDEX_NONE;
        if (bFound)
        {
            TickFunctionsToReschedule.RemoveAtSwap(Index);   
        }
        FTickFunction* PrevComparisonFunction = nullptr;
        FTickFunction* ComparisonFunction = AllCoolingDownTickFunctions.Head;
        while (ComparisonFunction && !bFound)
        {
            if (ComparisonFunction == TickFunction)
            {
                bFound = true;
                if (PrevComparisonFunction)
                {
                    PrevComparisonFunction->InternalData->Next = TickFunction->InternalData->Next;
                }
                else
                {
                    check(TickFunction == AllCoolingDownTickFunctions.Head);
                    AllCoolingDownTickFunctions.Head = TickFunction->InternalData->Next;
                }
                if (TickFunction->InternalData->Next)
                {
                    // haker: relative cool-down value is based on prev-TickFunction
                    // - the current TickFunction is going to be popped, so we add the removing TickFunction's RelativeTickCooldown to next-TickFunction
                    TickFunction->InternalData->Next->InternalData->RelativeTickCooldown += TickFunction->InternalData->RelativeTickCooldown;
                    TickFunction->InternalData->Next = nullptr;
                }
            }
            else
            {
                PrevComparisonFunction = ComparisonFunction;
                ComparisonFunction = ComparisonFunction->InternalData->Next;
            }
        }
        check(bFound); // otherwise you changed TickState while the tick function was registered. Call SetTickFunctionEnable instead.
        break;
    }
    // NewlySpawned -> 새로 추가되어 이번 프레임에는 Tick에 포함이 안된 상태 
    if (bTickNewlySpawned)
    {
        NewlySpawnedTickFunctions.Remove(TickFunction);
    }

    // haker: we safely removed FTickFunction from all queues in TickTaskLevel
}
```

## 설명
- [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]이 들고 있는 모든 큐에서 [[FTickFunction/FTickFunction|FTickFunction]]을 제거하는 로직. `AllEnabledTickFunctions`, `AllDisabledTickFunctions`, `TickFunctionsToReschedule`, `AllCoolingDownTickFunctions`가 서로 **동기화**되어야 하기 때문에 까다롭고 중요하다.
- `TickState`에 따라 세 갈래로 처리한다.
- `Enabled`인 경우:
  - `bWasInternal`이 false면 이미 tick 중이라는 뜻이므로 `AllEnabledTickFunctions`에서 바로 제거된다(`verify(... == 1)`).
  - `bWasInternal`이 true면 직전에 reschedule/cooling down 상태였을 수 있다. `AllEnabledTickFunctions.Remove()`가 0을 반환하면(= 삭제 실패, 아직 enabled 배열에 없음) `TickFunctionsToReschedule`([[FTickScheduleDetails/FTickScheduleDetails|FTickScheduleDetails]] 배열, 선형 탐색)을 먼저 뒤지고, 거기서 못 찾았을 때만 [[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|AllCoolingDownTickFunctions]] linked list를 순회한다.
  - linked list에서 중간 노드를 빼는 전형적인 처리다. head면 `Head`를 `Next`로 옮기고, 아니면 이전 노드의 `Next`를 이어 붙인다.
  - 둘 중 어디에서도 못 찾으면 `check(bFound)`로 걸린다.
- `Disabled`인 경우: 묻지도 따지지도 않고 `AllDisabledTickFunctions`에 있어야 한다.
- `CoolingDown`인 경우: `Enabled` + `bWasInternal` 경로와 같은 탐색을 하되, 노드를 빼면서 제거되는 tick function의 `RelativeTickCooldown`을 다음 노드에 더해준다. cooldown 값이 **이전 노드 기준의 상대값**이기 때문이다.
- 마지막으로 tick phase 중(`bTickNewlySpawned`)이라면 `NewlySpawnedTickFunctions`에서도 제거한다.
