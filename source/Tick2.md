---
summarize: true
---

#### Foundation - Tick - UWorld::SetupPhysicsTickFunctions(PhysLevel.cpp)

UWorld가 가지고 있는 StartPhysicsTickFunction, EndPhysicsTickFunction를 초기화하고 Level에 등록하는 작업을 수행

지금까지의 RegisterTickFunction, UnRegisterTickFunction의 반복

중요한 내용은 AddPrerequisite를 통해 FTickFunction 사이의 선후 관계를 설정하게 됨(EndPhysicsTickFunction의 Prerequisite로 StartPhysicsTickFunction를 설정)

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

#### Foundation - Tick - FTickFunction::IsTickFunctionRegistered(EngineBaseType.h)

```cpp
/** see if the tick function is currently registered */
bool IsTickFunctionRegistered() const { return (InternalData && InternalData->bRegistered); }
```

#### Foundation - Tick - class UWorld(World.h)

```cpp
/** tick function for starting physics */
FStartPhysicsTickFunction StartPhysicsTickFunction;
/** tick function for ending physics */
FEndPhysicsTickFunction EndPhysicsTickFunction;
```

#### Foundation - Tick - struct FPhysicsTickFunction(World.h)

```cpp
/** tick function that starts the physics tick */
struct FStartPhysicsTickFunction : public FTickFunction
{
    /** world this tick function belongs to */
    UWorld* Target;
};

/** tick function that ends the physics tick */
struct FEndPhysicsTickFunction : public FTickFunction
{
    /** world this tick function belongs to */
    UWorld* Target;
};
```

#### Foundation - Tick - FTickFunction::AddPrerequisite(TickTaskManager.cpp)

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

#### Foundation - Tick - struct FTickFunction(EngineBaseTypes.h)

```cpp
/** prerequisites for this tick function */
// haker: we can specify prerequisites for the tick function
TArray<FTickPrerequisite> Prerequisites;
```

#### Foundation - Tick - struct FTickPrerequisite(EngineBaseTypes.h)

```cpp
/** this is small structure to hold prerequisite tick functions */
struct FTickPrerequisite
{
		// PrerequisiteTickFunction이 유효한지는 PrerequisiteObject(이 경우 UWorld)가 valid한지로 판단
    /** tick functions live inside of UObjects, so we need a separate weak pointer to the UObject solely for the purpose of determining if PrerequisiteTickFunction is still valid */
    // haker: normally FTickFunction's aliveness is determined by UObject(ActorTickFunction -> AActor, ActorComponentTickFunction -> UActorComponent)
    TWeakObjectPtr<UObject> PrerequisiteObject;

    /** pointer to the actual tick function and must be completed prior to our tick running */
    FTickFunction* PrerequisiteTickFunction;

    FTickPrerequisite(UObject* TargetObject, struct FTickFunction& TargetTickFunction)
        : PrerequisiteObject(TargetObject)
        , PrerequisiteTickFunction(&TargetTickFunction)
    {}
};
```

#### **Foundation - Tick - enum ETickingGroup(EngineBaseTypes.h)**

Tick의 순서를 나누는 것도 맞지만 Task들이 병렬로 실행되는 과정에서 하나의 싱크 포인트(Join이 호출되는) 기점을 의미하기도 함
```cpp        
UENUM(BlueprintType)
enum ETickingGroup : int
{
    /** any item that needs to be executed before physics simulation starts */
    TG_PrePhysics UMETA(DisplayName="Pre Physics"),

    /** special tick group that start physics simulation */
    TG_StartPhysics UMETA(Hidden, DisplayName="Start Physics"),

    /** any item that can be run in parallel with our physics simulation work */
    // 이때 피직스 스레드가 병렬로 돌고 있기 때문에 이때 라인체크, 콜리전체크를 하게 되면 락이 걸려서 병목이 발생함
    // 그래서 이때는 주의해야 함
    TG_DuringPhysics UMETA(DisplayName="During Physics"),

    /** special tick group that ends physics simulation */
    TG_EndPhysics UMETA(Hidden, DisplayName="End Physics"),

    /** any item that needs rigid body and cloth simulation to be complete before being executed */
    TG_PostPhysics UMETA(DisplayName="Post Physics"),

    /** any item that needs the update work to be done before being ticked */
    TG_PostUpdateWork UMETA(DisplayName="Post Update Work"),

    /** catchall for anything demoted to the end */
    TG_LastDemotable UMETA(Hidden, DisplayName = "Last Demotable"),

    /** 
     * special tick group that is not actually a tick group
     * after every tick group, this is repeatedly re-run until there are no more newly spawned items to run
     */
    TG_NewlySpawned UMETA(Hidden, DisplayName="Newly Spawned"),

    TG_MAX,
};
```
**1. TG_PostUpdateWork**
- 물리 시뮬레이션(TG_EndPhysics/TG_PostPhysics)까지 다 끝난 "이후"에 실행
- 즉, 이번 프레임에 캐릭터/오브젝트의 최종 위치·회전이 물리엔진에 의해 확정된 뒤에 돌아가는 로직용
- 대표적으로 카메라(CameraComponent, SpringArm 등)가 여기에 해당함. 카메라는 캐릭터가 물리로 움직인 "최종 결과"를 보고 따라가야 하니까, 물리 업데이트보다 먼저 틱하면 한 프레임 밀린 것처럼 떨리거나 어긋나는 문제가 발생, 그래서 "업데이트 작업이 끝난 뒤에 틱해야 하는 애들"이 이 그룹에 포함됨

**2. TG_LastDemotable**
- 이름 그대로 "더 이상 밀 데가 없어서 맨 뒤로 강등된(demoted) 것들의 처리순서". 
- `Hidden`으로 되어 있어서 에디터에서 사용자가 직접 선택은 불가능
- 어떤 액터/컴포넌트가 다른 액터에 대해 "반드시 더 늦게 틱해야 해(AddTickPrerequisiteActor/Component 같은 의존성)"라는 제약을 걸었는데, 그 의존성 체인을 따라가다 보면 이미 정의된 마지막 정식 틱 그룹(TG_PostUpdateWork)보다도 더 뒤로 밀려야 하는 경우가 이 그룹에 할당.
- 개발자가 직접 지정하는 그룹이 아니라 사실상 엔진 내부 스케줄러가 쓰는 안전장치.

**3. TG_NewlySpawned**
- 한 프레임 동안 위의 정식 틱 그룹들(PrePhysics ~ LastDemotable)을 순서대로 다 돌리고 난 뒤, 그 과정에서 새로 스폰된 액터(예: 다른 액터의 Tick 도중 SpawnActor로 생성된 것)들을 처리하기 위한 그룹.
- 새로 스폰된 액터를 처리하다가 또 그 안에서 새 액터가 스폰될 수도 있으니, "더 이상 새로 스폰되는 게 없을 때까지" 반복적으로 재실행.
- 목적은 "이번 프레임에 늦게 태어난 액터도 같은 프레임 안에서 최소 한 번은 틱을 돌게 해주기" 위함. 


#### **Foundation - Tick - struct FTickFunction(EngineBaseTypes.h)**

TickGroup에도 Start와 End가 있어서 복수 개의 Group에 걸쳐있을 수 있음(그래프의 SpecialTickFunction)
![[Pasted image 20260819154106.png]]

```cpp
/**
 * defines the minimum tick group for this tick function
 * these groups determine the relative order of when objects tick during a frame update
 * - given prerequisites, the tick may be delayed 
 */
 
TEnumAsByte<enum ETickingGroup> TickGroup;

/** 
 * defines the tick group that this tick function must finished in
 * these tick group determine the relative order of when objects tick during a frame update
 */
TEnumAsByte<enum ETickingGroup> EndTickGroup;
```


#### **Foundation - Tick - struct FTickFunction::FInternalData(EngineBaseTypes.h)**
![[Pasted image 20260819155011.png]]

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
    // - for now, try to understand with Diagram concisely:
    // - Diagram:
    // - we'll see that:
    // 내 앞에 있는 노드 FTickFunction과 몇 초 차이나는지
    // 사용되는 곳은 sorting 관련?
    //   - cooling-down list is actually **sorted!** (how? will see soon!)
    float RelativeTickCooldown;
    
		// Tick 스케쥴링에 활용
    /** internal data to track if we have ***started*** visiting this tick function yet this frame */
    // haker: skip! TickVisitedGFrameCounter is related to support prerequisites! (you'll see how soon in the code)
    int32 TickVisitedGFrameCounter;
    
		// Tick 스케쥴링에 활용
    /** internal data to track if we have ***finished*** visiting this tick function yet this frame */
    // haker: skip!
    std::atomic<int32> TickQueuedGFrameNumber;
    
		// prerequisites에 따라 실제로 도는 tick 그룹이 바뀔 수도 있기 때문에 변수가 따로 있음
    /** internal data that indicates the tick group we actually started in (it may have been delayed due to prerequisites) */
    // haker: even though TickFunction has Start/EndTickGroup, depending on its prerequisites, tick group can be changed
    TEnumAsByte<enum ETickingGroup> ActualStartTickGroup;

    /** Internal data that indicates the tick group we actually started in (it may have been delayed due to prerequisites) **/
    TEnumAsByte<enum ETickingGroup> ActualEndTickGroup;
    
		// 실제 Tick을 실행하는데 사용되는 FTaskGraph의 포인터
    /** pointer to the task, only used during setup; this is often stale */
    // haker: the tick function is wrapped up by TickTaskManager and TickTaskSequencer, but it actually is run by FTaskGraph (we'll NOT cover this in our current course)
    FBaseGraphTask* TaskPointer;
};
```