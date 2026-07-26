---
related:
  - "[[ULevel/ULevel|ULevel]]"
  - "[[USceneComponent/USceneComponent|USceneComponent]]"
  - "[[UObject/UObject|UObject]]"
tags:
  - Actor_h
---

```cpp
class AActor : public UObject
{
		/** all ActorComponents owned by this Actor; stored as a Set as actors may have a large number of components */
		// haker: all components in AActor is stored here
		// 내부적으로 컴포넌트를 분류하는 별도의 컨테이너들도 존재하지만 모든 컴포넌트는 일단 이 안에 들어옴
		TSet<TObjectPtr<UActorComponent>> OwnedComponents;
		
		// 이 밑의 플래그는 어느 단계까지 초기화가 진행됬는지를 판별하는 변수
		// 1바이트에 1비트만 사용한다는 의미
		/**
		 * indicates that PreInitializeComponents/PostInitializeComponents has been called on this Actor
		 * prevents re-initializing of actors spawned during level startup 
		 */
		// haker: when PostInitializeComponents is called, bActorInitialized is set to 'true'
		uint8 bActorInitialized : 1;
		
		/** whether we've tried to register tick functions; reset when they are unregistered */
		// haker: tick function is registered
		uint8 bTickFunctionsRegistered : 1;
		
		/** enum defining if BeginPlay has started or finished */
		enum class EActorBeginPlayState : uint8
		{
		    HasNotBegunPlay,
		    BeginningPlay,
		    HasBegunPlay,
		};
		
		/** 
		 * indicates that BeginPlay has been called for this actor 
		 * set back to HasNotBegunPlay once EndPlay has been called
		 */
		// haker: this is VERY INTERESTING!
		// - uint8 bit assignment is working even on middle of uint8's enum type!
		// - ActorHasBegunPlay is updated by BeginPlay() or EndPlay() calls
		EActorBeginPlayState ActorHasBegunPlay : 2;
		
		/** 
		 * the component that defines the transform (location, rotation, scale) of this Actor in the world
		 * all other components must be attached to this one somehow
		 */
		// haker: AActor manages its components(UActorComponent) in a form of scene graph using USceneComponent
		// 컴포넌트들 사이의 계층 구조(상대 좌표)를 가능케하는 기준
		TObjectPtr<USceneComponent> RootComponent;

		/** The UChildActorComponent that owns this Actor */
		// haker: UChildActorComponent supports connection between Actor and Actor
		// - we only have Actor-ActorComponent connection, how to support Actor-Actor connection?
		// 액터와 액터 사이의 결합을 위해 사용됨, 액터와 액터는 계층 구조로 묶을 수 없는데 그런 한계를 극복하는 방법
		TWeakObjectPtr<UChildActorComponent> ParentComponent;
};
```

## 설명
- `AActor` == `UActorComponent`의 집합(Entity-Component 구조). networking에서는 state 전파와 RPC의 단위(replication의 단위)가 된다.
- 액터는 동적으로 스폰되는 개념이 아니라 월드(Outer)의 서브 오브젝트로 포함된 관계이므로, 레벨이 로딩될 때 이미 스폰되어 있다.
- 액터 초기화 순서:
  1. `UObject::PostLoad` — 레벨이 스트리밍되거나 로딩될 때 호출
  2. `UActorComponent::OnComponentCreated` — 액터를 구성하는 컴포넌트 생성 (생성이 먼저!)
  3. `AActor::PreRegisterAllComponents` — 등록 전 pre-event
  4. `AActor::RegisterComponent` — 여러 프레임에 걸친 incremental registration. 메인 월드(에디터/게임) 외에 render world(`FScene`), physics world(`FPhysScene`)에도 액터의 분신(state)을 추가하는 과정
  5. `AActor::PostRegisterAllComponents` — 등록 후 post-event
  6. `AActor::UserConstructionScript` — BP viewport로 추가한 SimpleConstructionScript가 아니라, BP ConstructionScript 이벤트로 추가한 컴포넌트를 생성
  7. `AActor::PreInitializeComponents` — 초기화 전 pre-event
  8. `UActorComponent::Activate` — 여기서 tick function이 등록된다 (중요)
  9. `UActorComponent::InitializeComponent` — 엔진이 별 처리를 하지 않고, 유저가 override해 원하는 코드를 실행하는 단계
  10. `AActor::PostInitializeComponents`
  11. `AActor::BeginPlay` — 이 이후부터 Tick이 돌아간다
- 즉 흐름은 [Component Creation] → [Component Register] → [Component Initialization] 순.
- `OwnedComponents`에는 모든 컴포넌트가 일단 들어온다(내부적으로 분류용 컨테이너가 따로 있어도).
- `bActorInitialized`, `bTickFunctionsRegistered`는 어느 단계까지 초기화가 진행됐는지 나타내는 비트 필드. `ActorHasBegunPlay`는 `uint8` enum에도 비트 폭 지정이 동작한다는 점이 흥미롭다.
- `RootComponent`는 트랜스폼의 기준이 되는 루트 노드로, 컴포넌트들의 계층 구조(상대 좌표)를 성립시킨다. `OwnedComponents`는 linear, `RootComponent`는 hierarchical 표현.
- `ParentComponent`는 이름과 달리, 액터끼리는 계층 구조로 묶을 수 없다는 한계를 `UChildActorComponent`로 극복하기 위한 역참조(weak ptr)다.
