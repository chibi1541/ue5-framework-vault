---
summarize: true
---

#### // 38 - Foundation - CreateWorld - UWorld::UpdateWorldComponents(World.cpp)

```cpp
/** updates world components like e.g. line batcher and all level components */
void UpdateWorldComponents(bool bRerunConstructionScripts, bool bCurrentLevelOnly, FRegisterComponentContext* Context = nullptr)
{
    if (!IsRunningDedicatedServer())
    {
  		for (TObjectPtr<ULineBatchComponent>& LineBatcher : LineBatchers)
			{
				// haker:
        // - ULineBatchComponent is herited from UPrimitiveComponent
        // - ULineBatchComponent can be seen as UActorComponent
        // - UWorld is NOT AActor, but has UActorComponent as its dynamic object for LineBatcher
        // - LineBatchers should be registered separately
        // 얘는 CDO에 들어가지 않는 오브젝트, 사실상 서브 오브젝트가 아님
				if (!LineBatcher)
				{
					// 여기 outer 지정도 없음
					LineBatcher = NewObject<ULineBatchComponent>();
					LineBatcher->bCalculateAccurateBounds = false;
				}
			}
	
			for (TObjectPtr<ULineBatchComponent>& LineBatcher : LineBatchers)
			{
				if (!LineBatcher->IsRegistered())
				{
					LineBatcher->RegisterComponentWithWorld(this, Context);
				}
			}
    }

    {
        for (int32 LevelIndex = 0; LevelIndex < Levels.Num(); ++LevelIndex)
        {
            ULevel* Level = Levels[LevelIndex];
            ULevelStreaming* StreamingLevel = FLevelUtils::FindStreamingLevel(Level);
            
            // update the level only if it is visible (or not a streamed level)
            if (!StreamingLevel || Level->bIsVisible)
            {
                Level->UpdateLevelComponents(bRerunConstructionScripts, Context);
                IStreamingManager::Get().AddLevel(Level);
            }
        }
    }

    const TArray<UWorldSubsystem*>& WorldSubsystems = SubsystemCollection.GetSubsystemArray<UWorldSubsystem>(UWorldSubsystem::StaticClass());
    for (UWorldSubsystem* WorldSubsystem : WorldSubsystems)
    {
        WorldSubsystem->OnWorldComponentsUpdated(*this);
    }
}
```

#### // 39 - Foundation - CreateWorld - UActorComponent::RegisterComponentWithWorld(ActorComponent.cpp)

LineBatcher 같은 World에 바로 종속된 ActorComponent를 Register 하기 위한 함수

```cpp
/** registers a component with a specific world, which creates any visual/physical state */
void RegisterComponentWithWorld(UWorld* InWorld, FRegisterComponentContext* Context = nullptr)
{
    // if the component was already registered, do nothing
    if (IsRegistered())
    {
        return;
    }

    // haker: it is natural to early-out cuz there is no world to register
    // 이 구문이 있다는 건 어딘가에 World가 생성되기 전에 Register가 호출 될 수도 있는 건가?
    if (InWorld == nullptr)
    {
        return;
    }

    // if not registered, should not have a scene
    // haker: it means that we can know whether it is registered or not by its existance of WorldPrivate
    check(WorldPrivate == nullptr);

    // haker: UWorld::LineBatcher(ULineBatchComponent)'s owner is nullptr
    AActor* MyOwner = GetOwner();
    checkSlow(MyOwner == nullptr || MyOwner->OwnsComponent(this));

    if (!HasBeenCreated)
    {
        // haker: do you remember OnComponentCreated() event covered in AActor?
        // - it is called when ActorComponent is being created
        OnComponentCreated();
    }

    WorldPrivate = InWorld;

    // 여기서 각 월드(랜더링, 피직스)의 분신(State)를 생성
    ExecuteRegisterEvents(Context);

    // if not in a game world register ticks now, otherwise defer until BeginPlay
    // if no owner we won't trigger BeginPlay() either so register now in that case as well
    if (!InWorld->IsGameWorld())
    {
        // haker: 
        // - if the world is not game world, we didn't call InitializeComponent()
        //   - see the condition below (MyOwner == nullptr) 
        RegisterAllComponentTickFunctions(true);
    }
    else if (MyOwner == nullptr)
    {
        // haker: here is the InitializeComponent() call which we covered in AActor's initialization stages
        if (!bHasBeenInitialized && bWantsInitializeComponent)
        {
            InitializeComponent();
        }

        RegisterAllComponentTickFunctions(true);

        // haker: here is the exact case matching UWorld::LineBatcher
    }
    else
    {
        // haker: this is **the NORMAL case** we expect
        MyOwner->HandleRegisterComponentWithWorld(this);
    }

    // if this is a blueprint created component and it has component children they can miss getting registered in some scenarios
    if (IsCreatedByConstructionScript())
    {
        // haker: explain what is SCS(Simple Construction Script) and CCS(Custom Construction Script) with examples (or with editor)
        // - Children will be collected from SCS and CCS, and they have outer object as this UActorComponent
        TArray<UObject*> Children;
        GetObjectsWithOuter(this, Children, true, RF_NoFlags, EInternalObjectFlags::Garbage);

        // haker: iterating Children, and try to RegisterComponentWithWorld (recursive calls)
        for (UObject* Child : Children)
        {
            if (UActorComponent* ChildComponent = Cast<UActorComponent>(Child))
            {
                if (ChildComponent->bAutoRegister && !ChildComponent->IsRegistered() && ChildComponent->GetOwner() == MyOwner)
                {
                    ChildComponent->RegisterComponentWithWorld(InWorld);
                }
            }
        }
    }
}

// 컴포넌트 생성 시에 무언가 처리를 넣고 싶으면 이걸 오버라이딩
/** called when a component is created (not loaded); this can happen in the editor or during gameplay */
virtual void OnComponentCreated()
{
    ensure(!bHasBeenCreated);
    bHasBeenCreated = true;
}
```

#### // 40 - Foundation - CreateWorld - UActorComponent::ExecuteRegisterEvents(ActorComponent.cpp)

```cpp
/** calls OnRegister, CreateRenderState_Concurrent and OnCreatePhysicsState */
void ExecuteRegisterEvents(FRegisterComponentContext* Context = nullptr)
{
    if (!bRegistered)
    {
        OnRegister();
    }

    if (FApp::CanEverRender() && !bRenderStateCreated && WorldPrivate->Scene 
        // haker: look ShouldCreateRenderState() 
        && ShouldCreateRenderState())
    {
        // haker:
        // - remember this
        // - this is the place that we add primitive to render world (world's scene == FScene)
        CreateRenderState_Concurrent(Context);
    }

    // haker:
    // we are not going to do deep-dive on physics for a while.
    CreatePhysicsState(/*bAllowDeferral=*/true);

    // haker:
    // - create[render|physics]state means 'creating the reflection of main world's state into each render-world and physics world'
}
```

#### // 41 - Foundation - CreateWorld - UActorComponent::OnRegister(ActorComponent.cpp)

```cpp
/** called when a component is registered, after Scene is set, but before CreateRenderState_Concurrent or OnCreatePhysicsState are called */
virtual void OnRegister()
{
    bRegistered = true;

    // haker: 
    // as the comment describes, this function is called before Create[Render|Phyiscs]State:
    // - it infers we should update all component transforms correctly
    // - if a component is USceneComponent, it should update all children's transforms
    UpdateComponentToWorld();

		// Register 단계에서 Activate를 수행
    // haker: when bAutoActivate is enabled, it try to activate on registeration stage
    if (bAutoActivate)
    {
        AActor* Owner = GetOwner();

        // haker: Owner == nullptr which is exact condition we matched for LineBatcher
        if (!WorldPrivate->IsGameWorld() 
            // haker: Owner can be nullptr, if a component is contained directly by UWorld
            // e.g. LineBatcher in UWorld
            || Owner == nullptr 
            || Owner->IsActorInitialized)
        {
            // haker: as we covered before, Activate() is usually called on BeginPlay(
            Activate(true);
        }
    }
}
```

#### // 42 - Foundation - CreateWorld - UActorComponent::Activate(ActorComponent.cpp)

```cpp
/** activates the SceneComponent, should be overriden by native child classes */
virtual void Activate(bool bReset=false)
{
    // haker: before we are getting into the details of Activate, see FTickFunction related classes first:
    if (bReset || ShouldActivate() == true)
    {
        SetComponentTickEnabled(true);
        
        SetActiveFlag(true);

        OnComponentActivated.Broadcast(this, bReset);
    }
}
```


# TickFunction Related Class

#### Foundation - CreateWorld - UActorComponent::SetComponentTickEnabled(ActorComponent.cpp)

```cpp
void UActorComponent::SetComponentTickEnabled(bool bEnabled)
{
	if (PrimaryComponentTick.bCanEverTick && !IsTemplate())
	{
		PrimaryComponentTick.SetTickFunctionEnable(bEnabled);
	}
}
```

#### Foundation - CreateWorld - struct FActorComponentTickFunction(EngineBaseTypes.h)

FActorComponentTickFunction, FActorTickFunction의 차이는 타겟이 AActor인지 UActorComponent 인지

```cpp
UPROPERTY(EditDefaultsOnly, Category="ComponentTick")
struct FActorComponentTickFunction PrimaryComponentTick;class UActorComponent : public UObject, public IInterface_AssetUserData, public IAsyncPhysicsStateProcessor
{
#if UE_WITH_REMOTE_OBJECT_HANDLE
	TObjectPtr<class UActorComponent> Target;
#else
	/** Actor component that is the target of this tick */
	class UActorComponent*	Target;
#endif
}

// UActorComponent의 FTickFunction
/** Main tick function for the Component */
UPROPERTY(EditDefaultsOnly, Category="ComponentTick")
struct FActorComponentTickFunction PrimaryComponentTick;
```

#### Foundation - CreateWorld - struct FActorTickFunction(EngineBaseTypes.h)

```cpp
USTRUCT()
struct FActorTickFunction : public FTickFunction
{
	/**  AActor  that is the target of this tick **/
#if UE_WITH_REMOTE_OBJECT_HANDLE
		TObjectPtr<AActor> Target;
#else
		class AActor*	Target;
#endif
};

// AActor의 FTickFunction
/**
 * Primary Actor tick function, which calls TickActor().
 * Tick functions can be configured to control whether ticking is enabled, at what time during a frame the update occurs, and to set up tick dependencies.
 * @see <https://docs.unrealengine.com/API/Runtime/Engine/Engine/FTickFunction>
 * @see AddTickPrerequisiteActor(), AddTickPrerequisiteComponent()
 */
UPROPERTY(EditDefaultsOnly, Category=Tick)
struct FActorTickFunction PrimaryActorTick;
```

#### Foundation - CreateWorld - struct FTickFunction(EngineBaseTypes.h)

```cpp
/** 
* Abstract Base class for all tick functions.
**/
USTRUCT()
struct FTickFunction
{
public:
	// The following UPROPERTYs are for configuration and inherited from the CDO/archetype/blueprint etc

	/**
	 * Defines the minimum tick group for this tick function. These groups determine the relative order of when objects tick during a frame update.
	 * Given prerequisites, the tick may be delayed.
	 *
	 * @see ETickingGroup 
	 * @see FTickFunction::AddPrerequisite()
	 */
	UPROPERTY(EditDefaultsOnly, Category="Tick", AdvancedDisplay)
	TEnumAsByte<enum ETickingGroup> TickGroup;
	
	/** If false, this tick function will never be registered and will never tick. Only settable in defaults. */
	UPROPERTY()
	uint8 bCanEverTick:1;
	
	/** If true, this tick function will start enabled, but can be disabled later on. */
	// haker: like bAutoActivate, when tick function is instantiated, automatically register tick function to tick every frame
	UPROPERTY(EditDefaultsOnly, Category="Tick")
	uint8 bStartWithTickEnabled:1;
	
	enum class ETickState : uint8
	{
		Disabled,
		Enabled,
		CoolingDown
	};

	/** 
	 * If Disabled, this tick will not fire
	 * If CoolingDown, this tick has an interval frequency that is being adhered to currently
	 * CAUTION: Do not set this directly
	 **/
	// haker: tick function could be specified its frequency
  // - when frequency value is set, tick function will have a moment to be in CoolingDown
  // Tick의 주기를 제어하는데 활용됨
  // 매 프레임 돌면 항상 Enabled, 특정 주기만큼 돈다면 돌지 않는 동안은 CoolingDown
  // 그나저나 모빌리티가 static이면 Tick도 안돌아?
	ETickState TickState : 2;
	
  /** internal data structure that contains members only required for a registered tick function */
  struct FInternalData
  {
      /** whether the tick function is registered */
      bool bRegistered : 1;

      /** cache whether this function was rescheduled as an interval function during StartParallel */
      // haker: if true, the TickFunction is in CoolingDown list
      // true면 쿨다운 중인거야?
      bool bWasInternal : 1;

      /** the next function in the cooling down list for ticks with an interval */
      // haker: cooling down tick-function list is in a form of linked list
      // 쿨링 다운 리스트를 내부적으로 링크드 리스트로 관리하는데 이걸로 링크드 리스트를 만든다고 보면 되나?
      // 언리얼에서 이런 방식으로 많이 쓴다고 함(리플렉션)
      FTickFunction* Next;

      /** back pointer to the FTickTaskLevel containing this tick function if it is registered */
      // haker: FTickFunction has similar relationship between AActor and ULevel:
      // - FTickFunction is contained in FTickTaskLevel
      // FTickFunction을 Level별로 모아놓은 군집군
      class FTickTaskLevel* TickTaskLevel;
  };                                                                                                                                                                                                                                                                                                  
  
	// Tick이 돌지 않아도 TickFunciton 자체는 만들어지기 때문에 Actor, ActorComponent만 생각해도
  // 게임 내부에 어마어마한 TickFunction이 있다는걸 예측할 수 있음
  // 이 때 Tick이 돌 필요가 없는 경우 FInternalData에 할당이 안돼 있는 방식으로 구성(hot/cold data)
  // 곧 Tick과 관련된 중요한 정보는 여기에 저장되어 있음
  /** lazily allocated struct that contains the necessary data for a tick function that is registered */
  // haker: why does it separate data as InternalData?
  // - we can achieve the following benefits, splitting hot/cold data
  //   - memory optimization
  //   - performance optimization
  // - think how many tick functions exist: Actor & ActorComponent
  TUniquePtr<FInternalData> InternalData;
};
```

#### Foundation - CreateWorld - enum ETickingGroup(EngineBaseTypes.h)

```cpp
/** Determines which ticking group a tick function belongs to. */
UENUM(BlueprintType)
enum ETickingGroup : int
{
	/** Any item that needs to be executed before physics simulation starts. */
	TG_PrePhysics UMETA(DisplayName="Pre Physics"),

	/** Special tick group that starts physics simulation. */							
	TG_StartPhysics UMETA(Hidden, DisplayName="Start Physics"),

	/** Any item that can be run in parallel with our physics simulation work. */
	TG_DuringPhysics UMETA(DisplayName="During Physics"),

	/** Special tick group that ends physics simulation. */
	TG_EndPhysics UMETA(Hidden, DisplayName="End Physics"),

	/** Any item that needs rigid body and cloth simulation to be complete before being executed. */
	TG_PostPhysics UMETA(DisplayName="Post Physics"),

	/** Any item that needs the update work to be done before being ticked. */
	TG_PostUpdateWork UMETA(DisplayName="Post Update Work"),

	/** Catchall for anything demoted to the end. */
	TG_LastDemotable UMETA(Hidden, DisplayName = "Last Demotable"),

	/** Special tick group that is not actually a tick group. After every tick group this is repeatedly re-run until there are no more newly spawned items to run. */
	TG_NewlySpawned UMETA(Hidden, DisplayName="Newly Spawned"),

	TG_MAX,
};
```

#### Foundation - CreateWorld - class FTickTaskLevel(TickTaskManager.cpp)

Actor 혹은 ActorComponent의 FTickFunction을 Level 단위로 관리하는 클래스

ULevel의 변수로 관리되며 생성자 호출 단계에서 생성 및 초기화가 가능(생성과 소멸의 주체는 ULevel, new로 생성되기 때문에 ULevel의 라이프 사이클로 제어)

```cpp
class FTickTaskLevel
{
		/** Global Sequencer*/
		// 이게 아마 Tick의 순서 제어 및 실행을 담당하는것 같은데 심지어 글로벌이라는 듯?
		FTickTaskSequencer&	TickTaskSequencer;

    /** primary list of enabled tick functions */
    // ETickState::Enable이 모여있는 TSet
    TSet<FTickFunction*> AllEnabledTickFunctions;

    /** primary list of enabled tick functions */
    // 이건 특이하게도 Linked List 구조(아마 CoolDown 시간에 따라 넣었다 뺐다 하기 때문인듯)
    // 이건 헤드만 보관하고 Next노드는 FInternalData.Next가 가리킴
    FCoolingDownTickFunctionList AllCoolingDownTickFunctions;

    /** primary list of disabled tick functions */
    TSet<FTickFunction*> AllDisabledTickFunctions;
    
    /** utility array to avoid memory reallocation when collecting functions to reschedule */
    // haker: FTickScheduleDetails is separate data for each tick-function to describe how it will be scheduled
    TArrayWithThreadsafeAdd<FTickScheduleDetails> TickFunctionsToReschedule;

    /** list of tick functions added during a tick phase; these items are also duplicated in AllLiveTickFunctions for future frames */
    TSet<FTickFunction*> NewlySpawnedTickFunctions;
    
    /** true during the tick phase, when true, tick function adds also go to the newly spawned list */
    bool bTickNewlySpawned;

    // haker: who is the owner of FTickTaskLevel to create/destroy?

    // haker: from here we can know two things:
    // 1. FTickFunction -> FTickTaskLevel:
    //   - we can see the abstraction between Actor,ActorComponent and TickFunction
    //   - Diagram:
    //          Level─────────────────────────────────►TickTaskLevel                                            
    //           │                                      │                                                       
    //           ├──Actor0────────────────────────────► ├──TickFunction0                                        
    //           │   │                                  │                                                       
    //           │   ├──ActorComponent0───────────────► ├──TickFunction1                                        
    //           │   │                                  │                                                       
    //           │   └──ActorComponent1───────────────► ├──TickFunction2                                        
    //           │                                      │                                                       
    //           └──Actor1────────────────────────────► ├──TickFunction3                                        
    //               │                                  │                                                       
    //               └──ActorComponent2───────────────► └──TickFunction4                                        
    //
    // 2. FTickFunction cycle:
    //   - Diagram: 
    //     haker: see this diagram just try to understand general pattern of tick-function life-cycle                                                                                                                                                                                                             
    //        ┌──TickFunction Cycle──────────────────────────────────────────────────┐                          
    //        │                          frequency is NOT set                        │                          
    //        │                        ┌───┐                                         │                          
    //        │                        │   │                                         │                          
    //        │                        │   │                                         │                          
    //        │    ┌──────────┐    ┌───┴───▼─┐    ┌─────────────┐    ┌─────────┐     │                          
    //        │    │ Register ├────► Enabled ├────► CoolingDown ├────► Disable │     │                          
    //        │    └──────────┘    └───▲─────┘    └──────┬──────┘    └─────────┘     │                          
    //        │                        │                 │                           │                          
    //        │                        └─────────────────┘                           │                          
    //        │                           frequency is set                           │                          
    //        │                                                                      │                          
    //        └──────────────────────────────────────────────────────────────────────┘                          
};

// ULevel의 FTickTaskLevel의 변수
ULevel::ULevel( const FObjectInitializer& ObjectInitializer )
	:	UObject( ObjectInitializer )
	,	Actors()
	,	OwningWorld(nullptr)
		// 여기가 FTickTaskLevel 변수 할당 부분
	,	TickTaskLevel(FTickTaskManagerInterface::Get().AllocateTickTaskLevel())
	,	PrecomputedLightVolume(new FPrecomputedLightVolume())
	,	PrecomputedVolumetricLightmap(new FPrecomputedVolumetricLightmap())
	,	RouteActorInitializationState(ERouteActorInitializationState::Preinitialize)
	,	RouteActorInitializationIndex(0)
	,   RouteActorEndPlayForRemoveFromWorldIndex(0)
{
}

ULevel::~ULevel()
{
	if (TickTaskLevel)
	{
		FTickTaskManagerInterface::Get().FreeTickTaskLevel(TickTaskLevel);
		TickTaskLevel = nullptr;
	}
}

// FTickTaskManager 함수
// 정말 할당밖에 하지 않음, 심지어 new로 생성됨
/** Allocate a new ticking structure for a ULevel **/
virtual FTickTaskLevel* AllocateTickTaskLevel() override
{
	return new FTickTaskLevel;
}
```

#### Foundation - CreateWorld - struct FCoolingDownTickFunctionList(TickTaskManager.cpp)

```cpp
// haker: see the member-variable, 'Head'
struct FCoolingDownTickFunctionList
{
    // 059 - Foundation - CreateWorld - FCoolingDownTickFunctionList::Contains
    bool Contains(FTickFunction* TickFunction) const
    {
        // haker: for junior, the data structure is important to understand the code
        FTickFunction* Node = Head;
        while (Node)
        {
            if (Node == TickFunction)
            {
                return true;
            }
            Node = Node->InternalData->Next;
        }
        return false;
    }

    // haker: we saw that InternalData->Next is the node for cooling-down list
    FTickFunction* Head;
};
```

#### Foundation - CreateWorld - struct FTickScheduleDetails(TickTaskManager.cpp)

이런 식으로 별도의 구조체를 통해 관리하는 이유는 쿨다운을 관리하는 매니저가 FTickFunction을 일일히 순회하면서 Cooldown을 체크하게 되면 FCoolingDownTickFunctionList(Linked List)의 각각 떨어져있는 Actor 혹은 ActorComponent를 순회하면서 데이터 조회를 해야해서 캐시히트율이 떨어지게 됨

그렇기 때문에 Tick을 스케쥴링 할 수 있는 정보들만 모아서 별도의 배열로 관리하는 듯함

```cpp
struct FTickScheduleDetails
{
    FTickFunction* TickFunction;
    // 쿨타임에 대한 정보
    float Cooldown;
    bool bDeferredRemove;
};
```