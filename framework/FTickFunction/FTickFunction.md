---
related:
  - "[[Enum/ETickingGroup|ETickingGroup]]"
  - "[[FActorComponentTickFunction/FActorComponentTickFunction|FActorComponentTickFunction]]"
  - "[[FActorTickFunction/FActorTickFunction|FActorTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]"
  - "[[FTickScheduleDetails/FTickScheduleDetails|FTickScheduleDetails]]"
  - "[[UActorComponent/UActorComponent.Activate|UActorComponent::Activate]]"
  - "[[UActorComponent/UActorComponent.SetComponentTickEnabled|UActorComponent::SetComponentTickEnabled]]"
  - "[[FTickFunction/FTickFunction.SetTickFunctionEnable|FTickFunction::SetTickFunctionEnable]]"
  - "[[FTickFunction/FTickFunction.IsTickFunctionRegistered|FTickFunction::IsTickFunctionRegistered]]"
  - "[[FTickFunction/FTickFunction.RegisterTickFunction|FTickFunction::RegisterTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel.RemoveTickFunction|FTickTaskLevel::RemoveTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel.AddTickFunction|FTickTaskLevel::AddTickFunction]]"
  - "[[FTickFunction/FTickFunction.AddPrerequisite|FTickFunction::AddPrerequisite]]"
  - "[[FTickFunction/FTickFunction.FInternalData|FTickFunction::FInternalData]]"
  - "[[FTickPrerequisite/FTickPrerequisite|FTickPrerequisite]]"
  - "[[FPhysicsTickFunction/FPhysicsTickFunction|FPhysicsTickFunction]]"
tags:
  - EngineBaseTypes_h
---

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

```cpp
/** prerequisites for this tick function */
// haker: we can specify prerequisites for the tick function
TArray<FTickPrerequisite> Prerequisites;
```

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

## 설명
![[Pasted image 20260819154106.png]]
- 모든 tick function의 추상 base 클래스. [[FActorTickFunction/FActorTickFunction|FActorTickFunction]], [[FActorComponentTickFunction/FActorComponentTickFunction|FActorComponentTickFunction]]이 이를 상속한다.
- `TickGroup`([[Enum/ETickingGroup|ETickingGroup]])은 이 tick function이 프레임 내 어느 시점에 실행되는지를 정하는 최소 tick group이다. prerequisite에 따라 더 늦게 실행될 수도 있다.
- `bCanEverTick`이 false면 tick function은 등록조차 되지 않는다. default에서만 설정 가능.
- `bStartWithTickEnabled`는 `bAutoActivate`와 비슷하게, 인스턴스화 시점에 매 프레임 tick 하도록 자동 등록할지를 정한다.
- `TickState`(`ETickState`)는 `Disabled` / `Enabled` / `CoolingDown` 세 가지다. tick 주기(frequency)를 제어하는 데 쓰이며, 매 프레임 돌면 항상 `Enabled`, 특정 주기로 돌면 쉬는 동안 `CoolingDown`이 된다. 직접 대입하면 안 된다.
- `FInternalData`는 **등록된** tick function에만 필요한 데이터를 모아둔 구조체이며 `TUniquePtr`로 lazy 할당된다. Actor/ActorComponent 수만큼 tick function이 만들어지므로, tick이 필요 없는 경우 이 데이터를 할당하지 않는 hot/cold data 분리로 메모리와 성능을 함께 최적화한다.
- `FInternalData::Next`는 cooling down 리스트를 linked list로 잇는 포인터다([[FCoolingDownTickFunctionList/FCoolingDownTickFunctionList|FCoolingDownTickFunctionList]]의 노드 역할).
- `FInternalData::TickTaskLevel`은 자신을 담고 있는 [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]로의 back pointer다. `AActor`와 `ULevel`의 관계와 비슷하게, tick function은 level 단위로 모여 관리된다.
- `Prerequisites`는 이 tick function보다 먼저 완료되어야 하는 tick function 목록이다([[FTickPrerequisite/FTickPrerequisite|FTickPrerequisite]] 배열). [[FTickFunction/FTickFunction.AddPrerequisite|AddPrerequisite()]]로 추가한다.
- `TickGroup`은 실행될 수 있는 **최소** tick group이고, `EndTickGroup`은 반드시 **끝나야 하는** tick group이다. 즉 하나의 tick function이 여러 group에 걸쳐 있을 수 있다.
- prerequisite 때문에 실제 실행 group이 밀릴 수 있어서, 실제로 시작/종료한 group은 [[FTickFunction/FTickFunction.FInternalData|FInternalData]]의 `ActualStartTickGroup` / `ActualEndTickGroup`에 따로 기록된다.
- `FInternalData`의 전체 멤버는 [[FTickFunction/FTickFunction.FInternalData|FTickFunction::FInternalData]] 문서에서 다룬다.
