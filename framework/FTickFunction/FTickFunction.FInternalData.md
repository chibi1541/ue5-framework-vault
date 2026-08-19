---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FTickFunction/FTickFunction.IsTickFunctionRegistered|FTickFunction::IsTickFunctionRegistered]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]"
  - "[[Enum/ETickingGroup|ETickingGroup]]"
tags:
  - EngineBaseTypes_h
---

```cpp
/** internal data structure that contains members only required for a registered tick function */
struct FInternalData
{
    /** whether the tick function is registered */
    bool bRegistered : 1;

    /** cache whether this function was rescheduled as an interval function during StartParallel */
    // haker: if true, the TickFunction is in CoolingDown list
    // - ***[TickTaskManager] it indicates whether it is in reschedule-list or cooling-down list (we'll see how it works soon)
    bool bWasInternal : 1;

    // 우선순위. TickTaskSequencer에 이걸 기준으로 Tick 배열이 나뉨
    /** run this tick first within the tick group, presumably to start async tasks that must be completed with this tick group, hiding the latency */
    // haker: TickTaskManager (actually TickTaskSequencer, more precisely FTaskGraph) maintain two queues(?) applying priority (normal + high-priority)
    uint8 bHighPriority : 1;

    /** if false, this tick will run on the game thread, otherwise it will run on any thread in parallel with the game thread and in parallel with other "async ticks" */
    // haker: if true, the tick function will run in task-thread (so running in parallel)
    // - by default(==false), it will run in game-thread
    uint8 bRunOnAnyThread : 1;

    /** the next function in the cooling down list for ticks with an interval */
    // haker: cooling down tick-function list is in a form of linked list
    FTickFunction* Next;

    /** back pointer to the FTickTaskLevel containing this tick function if it is registered */
    // haker: FTickFunction has similar relationship between AActor and ULevel:
    // - FTickFunction is contained in FTickTaskLevel
    class FTickTaskLevel* TickTaskLevel;

    /**
     * if TickFrequency is greater than 0 and tick state is CoolingDown, this is the time relative to
     * the element ahead of it in the cooling down list, remaining until the next time this function will tick
     */
    // haker: now we can understand what RelativeTickCooldown is in the code today
    // 내 앞에 있는 노드 FTickFunction과 몇 초 차이나는지
    //   - cooling-down list is actually **sorted!**
    float RelativeTickCooldown;

    // Tick 스케쥴링에 활용
    /** internal data to track if we have ***started*** visiting this tick function yet this frame */
    // haker: TickVisitedGFrameCounter is related to support prerequisites!
    int32 TickVisitedGFrameCounter;

    // Tick 스케쥴링에 활용
    /** internal data to track if we have ***finished*** visiting this tick function yet this frame */
    std::atomic<int32> TickQueuedGFrameNumber;

    // prerequisites에 따라 실제로 도는 tick 그룹이 바뀔 수도 있기 때문에 변수가 따로 있음
    /** internal data that indicates the tick group we actually started in (it may have been delayed due to prerequisites) */
    // haker: even though TickFunction has Start/EndTickGroup, depending on its prerequisites, tick group can be changed
    TEnumAsByte<enum ETickingGroup> ActualStartTickGroup;

    /** Internal data that indicates the tick group we actually started in (it may have been delayed due to prerequisites) **/
    TEnumAsByte<enum ETickingGroup> ActualEndTickGroup;

    // 실제 Tick을 실행하는데 사용되는 FTaskGraph의 포인터
    /** pointer to the task, only used during setup; this is often stale */
    // haker: the tick function is wrapped up by TickTaskManager and TickTaskSequencer, but it actually is run by FTaskGraph
    FBaseGraphTask* TaskPointer;
};
```

## 설명
![[Pasted image 20260819155011.png]]
- **등록된** tick function에만 필요한 데이터를 모아둔 구조체. [[FTickFunction/FTickFunction|FTickFunction]]이 `TUniquePtr`로 lazy 할당하므로, tick이 필요 없는 tick function은 이 데이터를 아예 갖지 않는다(hot/cold data 분리).
- `bRegistered`가 등록 여부의 실제 플래그이며, [[FTickFunction/FTickFunction.IsTickFunctionRegistered|IsTickFunctionRegistered()]]가 이 값을 본다.
- `bWasInternal`은 `StartParallel` 과정에서 interval tick으로 rescheduling 되었는지를 캐시한다. reschedule 리스트에 있는지 cooling down 리스트에 있는지를 구분하는 데 쓰인다.
- `bHighPriority`는 같은 tick group 안에서 먼저 실행할지를 정한다. TickTaskSequencer(내부적으로 `FTaskGraph`)가 normal / high-priority 두 개의 배열로 나누어 관리한다.
- `bRunOnAnyThread`가 true면 game thread가 아닌 task thread에서 병렬로 실행된다. 기본값 false는 game thread 실행이다.
- `Next`는 cooling down 리스트를 linked list로 잇는 포인터다([[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]의 노드 역할).
- `TickTaskLevel`은 자신을 담고 있는 [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]로의 back pointer다. `AActor`와 `ULevel`의 관계와 같다.
- `RelativeTickCooldown`은 cooling down 리스트에서 **바로 앞 노드와의 시간 차이**를 저장한다. 리스트가 정렬되어 있기 때문에 절대 시간이 아닌 상대 시간으로 관리한다.
- `TickVisitedGFrameCounter` / `TickQueuedGFrameNumber`는 이번 프레임에 이 tick function을 방문 **시작**했는지 / **완료**했는지를 추적하는 스케줄링용 데이터로, prerequisite 처리를 지원하기 위한 것이다.
- `ActualStartTickGroup` / `ActualEndTickGroup`은 실제로 실행된 tick group이다. `TickGroup`/`EndTickGroup`을 지정해도 prerequisite 때문에 뒤로 밀릴 수 있어 따로 기록한다([[Enum/ETickingGroup|ETickingGroup]]).
- `TaskPointer`는 실제 tick을 실행하는 `FTaskGraph` task로의 포인터다. setup 중에만 쓰이며 이후에는 stale한 값인 경우가 많다.
