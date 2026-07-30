---
related:
  - "[[FTickFunction/FTickFunction|FTickFunction]]"
  - "[[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]"
  - "[[FTickScheduleDetails/FTickScheduleDetails|FTickScheduleDetails]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[AActor/AActor|AActor]]"
tags:
  - TickTaskManager_cpp
---

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

## 설명
- `AActor` 혹은 `UActorComponent`의 [[FTickFunction/FTickFunction|FTickFunction]]을 [[ULevel/ULevel|ULevel]] 단위로 모아 관리하는 클래스. `AActor`와 `ULevel`의 관계가 `FTickFunction`과 `FTickTaskLevel`의 관계로 그대로 대응된다.
- `ULevel`의 멤버(`TickTaskLevel`)로 관리된다. 생성자에서 `FTickTaskManagerInterface::Get().AllocateTickTaskLevel()`로 할당하고 소멸자에서 `FreeTickTaskLevel()`로 해제하므로, 생성/소멸의 주체는 `ULevel`이고 라이프사이클도 `ULevel`을 따른다. `AllocateTickTaskLevel()`은 `new FTickTaskLevel` 뿐이다.
- tick function을 상태별로 나눠 보관한다:
  - `AllEnabledTickFunctions`: `ETickState::Enabled`인 것들의 `TSet`.
  - `AllCoolingDownTickFunctions`: [[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]. 특이하게 linked list 구조로, head만 보관하고 다음 노드는 `FTickFunction::FInternalData::Next`가 가리킨다. cooldown 시간에 따라 넣고 빼기 때문으로 보인다.
  - `AllDisabledTickFunctions`: disabled인 것들의 `TSet`.
- `TickFunctionsToReschedule`은 reschedule 대상을 모을 때 메모리 재할당을 피하기 위한 utility 배열이며, 원소는 [[FTickScheduleDetails/FTickScheduleDetails|FTickScheduleDetails]]다.
- `NewlySpawnedTickFunctions`는 tick phase 도중 추가된 tick function 목록이다. `bTickNewlySpawned`가 true인 동안(= tick phase 중) 추가되는 tick function은 이 리스트에도 함께 들어간다.
- `TickTaskSequencer`는 global sequencer로, tick 순서 제어 및 실행을 담당한다.
- tick function 생애주기: `Register` → `Enabled` → (frequency가 설정된 경우) `CoolingDown` ↔ `Enabled` → `Disable`. frequency가 설정되지 않으면 `Enabled`에 머문다.
