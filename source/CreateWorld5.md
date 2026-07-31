---
summarize: true
---

#### Foundation - CreateWorld - UActorComponent::SetComponentTickEnabled(ActorComponent.cpp)

```cpp
/** set this component's tick functions to be enabled or disabled; only has an effect if the function is registered */
virtual void SetComponentTickEnabled(bool bEnabled)
{
    // haker: bCanEverTick is the variable to determine whether tick-function is enabled for ticking every frame
    if (PrimaryComponentTick.bCanEverTick && !IsTemplate())
    {
        PrimaryComponentTick.SetTickFunctionEnable(bEnabled);
    }
}
```

#### What is IsTemplate?

#### Foundation - CreateWorld - UObjectBaseUtility::IsTemplate (UObjectBaseUtility.cpp)

```cpp
// 실제 인스턴스화 된 것이 아닌, 템플릿화 된 더미를 의미
// CDO는 기본적으로 RF_ArchetypeObject을 함께 갖는다고 함
// 그 밖에도 BP 클래스의 컴포넌트 템플릿이 RF_ArchetypeObject, 이 경우는 CDO는 아님
// Archetype
/** determine whether this object is a template object */
// haker: 
// - I have been look through the unreal engine source code for long time, but I still can't explain what is archetype object with specific example
// - you just think of it as CDO, class default object
//   - for CDO, what I understand is like **initialization list** as default UObject instance
bool IsTemplate(EObjectFlags TemplateTypes = RF_ArchetypeObject|RF_ClassDefaultObject) const
{
		// ULevel까지 올라가면서 체크, 이중에 CDO 같은 애들이 하나만 걸려도 Template 판정
		// CDO가 가지고 있는 서브 오브젝트가 전부 CDO가 아닐 수 있음
    // haker: note that if one of outer is template, the object is template
    for (const UObjectBaseUtility* TestOuter = this; TestOuter; TestOuter = TestOuter->GetOuter())
    {
        if (TestOuter->HasAnyFlags(TemplateTypes))
            return true;
    }
    return false;
}
```

#### Foundation - CreateWorld - FTickFunction::SetTickFuncitonEnable (TickTaskManager.cpp)

TickFunction을 활성화, 비활성화 시키는 함수

지금 흐름도 월드에 액터 컴포넌트를 연결하는 일반적이지 않은 흐름이기 때문에

이런 과정에서 같은 처리를 2번 반복하지 않도록 하기 위해 언리얼은 클래스 내부적으로 상태 체크하는 플래그 변수를 두고 이를 확인하면서 로직을 수행하는 패턴을 많이 사용함

```cpp
void SetTickFunctionEnable(bool bInEnabled)
{
    // haker: this function is to enable self(tick-function) to tick()
    if (IsTickFunctionRegistered())
    {
        // haker: carefully read the condition in the if statement
        // - state is changed from disabled -> enabled
        if (bInEnabled == (TickState == ETickState::Disabled))
        {
            // haker: InternalData has tick-task-level which it is resides in
            FTickTaskLevel* TickTaskLevel = InternalData->TickTaskLevel;

						// 무조건 remove를 먼저 한번 함
            TickTaskLevel->RemoveTickFunction(this);
            TickState = (bInEnabled ? ETickState::Enabled : ETickState::Disabled);

            TickTaskLevel->AddTickFunction(this);

            // haker: from here, you should see simple-rule in coding:
            // when Enable/Disable FTickFunction, just ***re-insert it***
            // - it doesn't add any complex conditions to handle each case
        }

        if (TickState == ETickState::Disabled)
        {
            InternalData->LastTickGameTimeSeconds = -1.f;
        }
    }
    else
    {
        // haker: if it is NOT registered yet, just update the TickState
        // - when TickFunction is registered, it will handle it as we covered above
        TickState = (bInEnabled ? ETickState::Enabled : ETickState::Disabled);
    }
}
```

#### Foundation - CreateWorld - FTickFunction::IsTickFunctionRegistered (EngineBaseTypes.h)

```cpp
bool IsTickFunctionRegistered() const 
{
	return (InternalData && InternalData->bRegisterd);
}
```

#### Foundation - CreateWorld - FTickTaskLevel::RemoveTickFunction(TickTaskManager.cpp)

FTickTaskLevel에서 FTickFunction을 제거하는 로직

이 부분이 까다롭고 중요함

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

#### Foundation - CreateWorld - FTickTaskLevel::RemoveTickFunction(TickTaskManager.cpp)

Remove 과정에서 경우의 수에 따른 예외 처리를 마쳤기 때문에 추가하는건 단순

```cpp
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
        check(TickFunction->TickState == FTickFunction::ETickState::Disabled);
        AllDisabledTickFunctions.Add(TickFunction);
    }
}
```

#### Foundation - CreateWorld - FTickTaskLevel::HasTickFunction(TickTaskManager.cpp)

FCoolingDownTickFunctionList의 `Contains()`함수는 무한 루프 돌면서 노드들을 순회하는 전형적인 Linked List 순회 방식으로 구현되어 있음

```cpp
bool HasTickFunction(FTickFunction* TickFunction)
{
	return AllEnabledTickFunctions.Contains(TickFunction) || AllDisabledTickFunctions.Contains(TickFunction) || AllCoolingDownTickFunctions.Contains(TickFunction);
}
```

#### Foundation - CreateWorld - UActorComponent::RegisterAllComponentTickFunctions(ActorComponent.cpp)

```cpp
// bRegister = true
void RegisterAllComponentTickFunctions(bool bRegister)
{
    // components don't have tick functions until they are registered with the world
    // haker: bRegistered becomes true when OnRegister() is called (== registered with the world)
    // OnRegister가 호출되면 true가 됨
    // 그 말은 다른 월드에 state가 만들어진 후에야 TickFunction 등록이 된다는 의미
    if (bRegistered)
    {
        // prevent repeated redundant attempts
        if (bTickFunctionsRegistered != bRegister)
        {
            // 얘를 오버라이드 하는게 가능
            RegisterComponentTickFunctions(bRegister);
            bTickFunctionsRegistered = bRegister;
        }
        
        if (bAsyncPhysicsTickEnabled)
        {
            // haker: skip AsyncPhysicsTickEnabled():
            // - this is the TickFunction as we covered that multiple tick functions can be registered
            // 이건 단순히 TickFunction이 하나의 Actor, Component에 복수개 있을 수 있다 정도만 인식
            RegisterAsyncPhysicsTickEnabled(bRegister);
        }
    }
}
```

### Foundation - CreateWorld - UActorComponent::RegisterComponentTickFunctions(ActorComponent.cpp)

사용자가 오버라이딩하는 것이 가능

```cpp
virtual void RegisterComponentTickFunctions(bool bRegister)
{
    if (bRegister)
    {
        if (SetupActorComponentTickFunction(&PrimaryComponentTick))
        {
            PrimaryComponentTick.Target = this;
        }
    }
    else
    {
        if (PrimaryComponentTick.IsTickFunctionRegistered())
        {
            PrimaryComponentTick.UnRegisterTickFunction();
        }
    }
}
```

### Foundation - CreateWorld - UActorComponent::RegisterComponentTickFunctions(ActorComponent.cpp)

FTickFunction를 인자로 받아서 Tick을 Enable/Disable하는 함수

여기서는 앞선 TickFunction의 등록 과정 중 보았던 SetTickFunctionEnable 과 RegisterTickFunction(ULevel* InLevel) 를 함께 호출

```cpp
bool SetupActorComponentTickFunction(FTickFunction* TickFunction)
{
    if (TickFunction->bCanEverTick && !IsTemplate())
    {
        AActor* MyOwner = GetOwner();
        if (!MyOwner || !MyOwner->IsTemplate())
        {
		        // ActorComponent는 월드는 캐싱하지만 Level은 직접 캐싱하지 않음
            ULevel* ComponentLevel = (MyOwner ? MyOwner->GetLevel() : ToRawPtr(GetWorld()->PersistentLevel));
            // haker: if TickFunction is not registered, just TickState will be updated
            // - we have seen SetTickFunctionEnable(), but let's see one more again briefly
            // 여기서 TickFunction을 FTickTaskLevel에 등록
            TickFunction->SetTickFunctionEnable(TickFunction->bStartsWithTickEnabled || TickFunction->IsTickFunctionEnabled());

            // haker: we need ULevel where the component resides in, cuz FTickFunction is resides in TickTaskLevel!
            TickFunction->RegisterTickFunction(ComponentLevel);
            return true;
        }
    }
    return false;

    // haker: SetupActorComponentTickFunction makes sure that TickFunction is registered
}
```

### Foundation - CreateWorld - FTickFunction::RegisterTickFunction(TickTaskManager.cpp)

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

#### Foundation - CreateWorld - FTickTaskManager::AddTickFunction(TickTaskManager.cpp)

```cpp
void AddTickFunction(ULevel* InLevel, FTickFunction* TickFunction)
{
    FTickTaskLevel* Level = TickTaskLevelForLevel(InLevel);
    // 아 여기서 Level에 AddTickFunction을 호출함
    Level->AddTickFunction(TickFunction);
    TickFunction->InternelData->TickTaskLevel = Level;
}
```

#### Foundation - CreateWorld - FTickTaskManager::TickTaskLevelForLevel(TickTaskManager.cpp)

```cpp
FTickTaskLevel* TickTaskLevelForLevel(ULevel* Level, bool bCreateIfNeeded = true)
{
	check(Level);

	if (bCreateIfNeeded && Level->TickTaskLevel == nullptr)
	{
		// 레벨에 TickTaskLevel이 없는 경우 여기서 할당
		// AllocateTickTaskLevel() => return new FTickTaskLevel;
		Level->TickTaskLevel = AllocateTickTaskLevel();
	}

	check(Level->TickTaskLevel);
	return Level->TickTaskLevel;
}
```

#### Foundation - CreateWorld - FTickTaskLevel::AddTickFunction(TickTaskManager.cpp)

```cpp
/** Add the tick function to the primary list **/
void AddTickFunction(FTickFunction* TickFunction)
{
	check(!HasTickFunction(TickFunction));
	if (TickFunction->TickState == FTickFunction::ETickState::Enabled)
	{
		AllEnabledTickFunctions.Add(TickFunction);
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

#### Foundation - CreateWorld - AActor::HandleRegisterComponentWithWorld(Actor.cpp)

UWorld에서 바로 컴포넌트 Register 함수를 호출했으면 MyOnwer(AActor)가 없어서 이게 호출되지 않음

Owner(여기서는 Actor)의 상태에 따라서(Actor가 어느 정도 초기화가 진행되었는지) Component의 초기화 정도가 달라지기 때문에 Actor쪽 함수로 Component의 Register가 진행됨

여기가 UWorld → ULevel → AActor → UActorComponent 구조의 일반적인 처리 로직

Register단계 + 필요하다면 BeginPlay()까지 한번에 호출

```cpp
/** finish initializing the component and register tick functions and BeginPlay() if it's the proper time to do so */
void HandleRegisterComponentWithWorld(UActorComponent* Component)
{
		// ActorComponent가 초기화 단게에서만 생성, 등록되는 건 아니기 때문에
    const bool bOwnerBeginPlayStarted = HasActorBegunPlay() || IsActorBeginningPlay();

    // haker: if component is not initialized, try to initialize the component
    if (!Component->HasBeenInitialized() && Component->bWantsInitializeComponent && IsActorInitialized())
    {
		    // ActorComponent의 InitializeComponent 함수는 사용자가 오버라이드해서 커스텀하는 것을 주목적으로 설계된 함수
        Component->InitializeComponent();

        // the component was finally initialized, it can now be replicated
        // NOTE that if this component does not ask to be initialized, it would have started to be replicated inside AddOwnedComponent 
        // haker: in editor, if we don't play in PIE, BeginPlay() isn't called
        if (bOwnerBeginPlayStarted)
        {
		        // 언리얼 네트워크에서 RPC의 단위가 Actor이기 때문에
		        // 여기서 뭔가 처리를 해줌
            // haker: we skip the replication stuff for now
            AddComponentForReplication(Component);
        }
    }

    if (bOwnerBeginPlayStarted)
    {
        // haker: when are we getting into this code?
        // - maybe actor is activated, and it will be triggered, if we add new component to actor which is already activated
        Component->RegisterAllComponentTickFunctions(true);
        if (!Component->HasBegunPlay())
        {
		        // 이건 나중에
            Component->BeginPlay();
        }
    }
}
```

#### Foundation - CreateWorld - UActorComponent::IsCreatedByConstructionScript(ActorComponent.cpp)

```cpp
// ActorComponent의 멤버 변수
EComponentCreationMethod CreationMethod;

/** returns true if instances of this component are created by either the user or simple construction script */
bool IsCreatedByConstructionScript() const
{
    return ((CreationMethod == EComponentCreationMethod::SimpleConstructionScript) || (CreationMethod == EComponentCreationMethod::UserConstructionScript));
}
```

#### Foundation - CreateWorld - enum EComponentCreationMethod(ComponentInstanceDataCache.h)

- Native ⇒ C++ 코드에 의해 생성됨
- SimpleConstructionScript ⇒ BP의 Hierarchy 창에서 추가됨
- UserConstructionScript ⇒ UserConstructionScript 함수(BP의 ConstructionScript 이벤트) 내부에서 생성된 컴포넌트
- Instance ⇒ Actor의 Detail 패널에서 추가된 컴포넌트 인스턴스

```cpp
UENUM()
enum class EComponentCreationMethod : uint8
{
	/** A component that is part of a native class. */
	Native,
	/** A component that is created from a template defined in the Components section of the Blueprint. */
	SimpleConstructionScript,
	/**A dynamically created component, either from the UserConstructionScript or from a Add Component node in a Blueprint event graph. */
	UserConstructionScript,
	/** A component added to a single Actor instance via the Component section of the Actor's details panel. */
	Instance,
};

```