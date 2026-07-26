---
summarize: true
---

#### // 0 - Foundation - CreateWorld - BEGIN(UnrealEngine.cpp)

```cpp
void UEngine::Init(IEngineLoop* InEngineLoop)
{
	if (GIsEditor)
	{
		// Create a WorldContext for the editor to use and create an initially empty world.
		FWorldContext &InitialWorldContext = CreateNewWorldContext(EWorldType::Editor);
		InitialWorldContext.SetCurrentWorld( UWorld::CreateWorld( EWorldType::Editor, true ) );
		GWorld = InitialWorldContext.World();
	}
}
```


#### // 1 - Foundation - CreateWorld - UWorld::CreateWorld(World.cpp)

```cpp
using InitializationValues = FWorldInitializationValues;

// haker: note that this function is static function
// - when you look through parameters in a function, care about their types
// - when we use UWorld::CreateWorld(), we only pass two parameters, so rest of parameters is not necessary to care
static UWorld* CreateWorld(
    const EWorldType::Type InWorldType, 
    bool bInformEngineOfWorld, 
    FName WorldName = NAME_None, 
    UPackage* InWorldPackage = NULL, 
    bool bAddToRoot = true, 
    ERHIFeatureLevel::Type InFeatureLevel = ERHIFeatureLevel::Num, 
    // 월드를 만들 때 중요한 변수들을 모아놓은 플래그 변수
    const InitializationValues* InIVS = nullptr, 
    bool bInSkipInitWorld = false
)
{
    // haker: UPackage will be covered in the future, dealing with AsyncLoading
    // - for now, just think of it as describing file format for UWorld
    // - one thing to remember is that each world has 1:1 mapping on separate package
    //   - it is natural that all data in world needs to be serialized as file
    //   - in unreal engine, saving file means 'package', UPackage
    // - another thing is that UObject has OuterPrivate:
    //   - OuterPrivate infers where object is resides in
    //   - normally OuterPrivate is set as UPackage: 
    //     - where object is resides in == what file object resides in
    // - I'd like to explain what I understand about UPackage, don't memorize it, just say 'ah' is enough!
    // UPackage : 월드를 저장하고 있는 하나의 Asset 파일이라는 느낌
    // 저장을 위해 시리얼라이즈화 한 데이터
    UPackage* WorldPackage = InWorldPackage;
    if (!WorldPackage)
    {
        // haker: UWorld needs package as its OuterPrivate and need to be serialized
        WorldPackage = CreatePackage(nullptr);
    }

    if (InWorldType == EWorldType::PIE)
    {
        // haker: like ObjectFlags in UObjectBase, UPackage's attribute can be set by flags in similar manner
        // - we are not going to read it in detail, later we could have chance to meet again
        WorldPackage->SetPackageFlags(PKG_PlayInEditor);
    }

    // mark the package as containing a world
    // haker: what is 'Transient' in Unreal Engine?
    // - if you have property or asset which is not serialized, we mark it as 'Transient'
    // - Transient == (meta data to mark it as not-to-be-serialize)
    // Transient => 직렬화 하지 않겠다는 의미
    if (WorldPackage != GetTransientPackage())
    {
        WorldPackage->ThisContainsMap();
    }

    const FString WorldNameString = (WorldName != NAME_None) ? WorldName.ToString() : TEXT("Untitled");

    // haker: we set NewWorld's outer as WorldPackage:
    // - normally, when you look into outer object, it will finally end-up-with package(asset file containing this UObject)
    UWorld* NewWorld = NewObject<UWorld>(WorldPackage, *WorldNameString);

    // haker: UObject::SetFlags -> set unreal object's attribute with flag by bit operator (AND(&), OR(|), SHIFT(>>, <<) etc...)
    // - refer to EObjectFlags and UObject::ObjectFlags
    NewWorld->SetFlags(RF_Transactional);
    NewWorld->WorldType = InWorldType;
    NewWorld->SetFeatureLevel(InFeatureLevel);
    NewWorld->InitializeNewWorld(
        InIVS ? *InIVS : UWorld::InitializationValues()
            // haker: as we saw FWorldInitializationValues, the below member functions mark the flag to refer when we create new world
            .CreatePhysicsScene(InWorldType != EWorldType::Inactive)
            .ShouldSimulatePhysics(false)
            .EnableTraceCollision(true)
            .CreateNavigation(InWorldType == EWorldType::Editor)
            .CreateAISystem(InWorldType == EWorldType::Editor)
        , bInSkipInitWorld);
  
    // clear the dirty flags set during SpawnActor and UpdateLevelComponents
    WorldPackage->SetDirtyFlag(false);

    if (bAddToRoot)
    {
        // add to root set so it doesn't get GC'd
        NewWorld->AddToRoot();
    }

    // tell the engine we are adding a world (unless we are asked not to)
    if ((GEngine) && (bInformEngineOfWorld == true))
    {
        GEngine->WorldAdded(NewWorld);
    }

    return NewWorld;
}

```

#### // 3 - Foundation - CreateWorld - struct FWorldInitializationValues(WorldInitializationValues.h)

```cpp
/** struct containing a collection of optional parameters for initialization of a world */
// haker: think of this pattern as one struct encapsulating multiple parameters for the code readability
// - this struct contains all necessary options to create world
struct FWorldInitializationValues
{
    /** should the scenes (physics, rendering) be created */
    // haker: whether we create worlds (render world, physics world, ...)
    // Scene은 FScene을 의미
    uint32 bInitializeScenes:1;

    /** Should the physics scene be created. bInitializeScenes must be true for this to be considered. */
    uint32 bCreatePhysicsScene:1;

    /** Are collision trace calls valid within this world. */
    uint32 bEnableTraceCollision:1;

    //...
};
```

</aside>

<aside>  
📝

## UWorld

#### // 4 - Foundation - CreateWorld - class UWorld(World.h)

```cpp
/** 
 * the world is the top level object representing a map or a sandbox in which Actors and Components will exist and be rendered 
 * 
 * a world can be a single persistent level with an optional list of streaming levels that are loaded and unloaded via volumes and blueprint functions
 * or it can be a collection of levels organized with a World Composition (->haker: OLD COMMENT...)
 * 
 * in a standalone game, generally only a single World exists except during seamless area transition when both a destination and current world exists
 * in the editor many Worlds exist: 
 * - the level being edited
 * - each PIE instance
 * - each editor tool which has an interactive rendered viewport, and many more
 */
 
class UWorld final : public UObject, public FNetworkNotify
{
		/** the URL that was used when loading this World */
		// haker: think of the URL as package path:
		// - e.g. Game\Map\Seoul\Seoul.umap
		// 파일 Path, 패키지 Path
		FURL URL;
		
		/** the type of world this is. Describes the context in which it is being used (Editor, Game, Preview etc.) */
		// haker: we already seen EWorldType
		// - TEnumAsByte is helper wrapper class to support bit operation on enum type
		// - I recommend to read it how it is implemented
		// ADVICE: as C++ programmer, it is VERY important **to manipulate bit operations freely!**
		// enum 값을 비트 연산하기 위한 랩핑
		TEnumAsByte<EWorldType::Type> WorldType;
		
		/** persistent level containing the world info, default brush and actors pawned during gameplay among other things */
		// hacker: short explanation about world info
		TObjectPtr<class ULevel> PersistentLevel;
		// World Info는 에디터에서 World Setting에서 나오는 세팅 정보들을 의미
		// 이걸 가지고 있는게 PersistentLevel 그렇지 않은 레벨들은 SubLevel
}
```

![image.png](attachment:a95874cf-2c52-4967-bc28-e65527441b57:513d15ea-d92d-4cee-90f5-f6a379254377.png)

- 잠깐 UObject 확인
    
    <aside>  
    📝
    
    ### UObject
    
    UObject → UObjectBaseUtility → UObjectBase
    
    여기서 주로 볼 내용은 UObjectBase의 ObjectFlags와 OuterPrivate
    
    #### // 5 - Foundation - CreateWorld - class UObject(Object.h)
    
    ```cpp
    /**
     * the base class of all UE objects. the type of an object is defined by its UClass
     * this provides support functions for creating and using objects, and virtual functions that should be overriden in child classes
     */
     class UObject : public UObjectBaseUtility
    {
        
    };
    ```
    
    #### // 6 - Foundation - CreateWorld - class UObjectBaseUtility(UObjectBaseUtility.h)
    
    ```cpp
    /** provides utility function for UObject, this class should not be used directly */
    // haker: later we'll cover UObjectBaseUtility's member functions below
    class UObjectBaseUtility : public UObjectBase
    {
        UObject* GetTypedOuter(UClass* Target) const
        {
            UObject* Result = NULL;
            for (UObject* NextOuter = GetOuter(); Result == NULL && NextOuter != NULL; NextOuter = NextOuter->GetOuter())
            {
                // haker: we are not getting into IsA(), which is out-of-scope cuz it is related to Reflection System
                if (NextOuter->IsA(Target))
                {
                    Result = NextOuter;
                }
            }
            return Result;
        }
    
        /** traverses the outer chain searching for the next object of a certain type (T must be derived from UObject) */
        template <typename T>
        T* GetTypedOuter() const
        {
            return (T*)GetTypedOuter(T::StaticClass());
        }
    
        /** determine whether this object is a template object */
        // haker: 
        // - I have been look through the unreal engine source code for long time, but I still can't explain what is archetype object with specific example
        // - you just think of it as CDO, class default object
        //   - for CDO, what I understand is like **initialization list** as default UObject instance
        bool IsTemplate(EObjectFlags TemplateTypes = RF_ArchetypeObject|RF_ClassDefaultObject) const
        {
            // haker: note that if one of outer is template, the object is template
            for (const UObjectBaseUtility* TestOuter = this; TestOuter; TestOuter = TestOuter->GetOuter())
            {
                if (TestOuter->HasAnyFlags(TemplateTypes))
                    return true;
            }
            return false;
        }
    ```
    
    #### // 7 - Foundation - CreateWorld - class UObjectBase(UObjectBase.h)
    
    ```cpp
    /** low level implementation of UObject, should not be used directly in game code */
    // haker: this class is the most base class for UObject
    // - look through its member variables
    class UObjectBase
    {
        /**
         * Flags used to track and report various object states
         * this needs to be 8 byte aligned on 32-bit platforms to reduce memory waste 
         */
        // haker: bit flags to define UObject's behavior or attribute as meta-data format
        EObjectFlags ObjectFlags;
    
    		// GetOuter()를 호출했을 때 나오는 녀석
    		// UObject는 트리 구조로 구성됨
    		// UPackage (패키지)
        //  └── UWorld (월드)
        //        └── ULevel (레벨)
        //                └── AActor (액터)
        //                        └── UActorComponent (컴포넌트)
        //                                └── UObject (서브오브젝트)
        /** object this object resides in */
        // haker: as we said previously, it is written as UPackage
        // - note that as times went by, the unreal supports lots of features to support reduce dependency on assets:
        //   - OFPA (One File Per Actor) is one of representative example
        //   - in the past, AActor resides in ULevel and its UPackage is just level asset file, which is straight-forward
        //   - but, after introducing OFPA, an indirection is added, no more each AActor is stored in ULevel file, it is stored in separate file Exteral path
        // - I'd like to say that overall pattern is maintained, but as engine evolves, it adds indirection and complexity to understand its actual behavior
        // - anyway for now, you just try to understand OuterPrivate will be set as UPackage normally, it is enought for now!
        // OFPA 설정을 사용하게 되면 액터는 Level의 종속되지 않고 하나의 UPackage로 독립됨
        
    		ObjectPtr_Private::TNonAccessTrackedObjectPtr<UObject>						OuterPrivate;
    };
    ```
    
    #### // 8 - Foundation - CreateWorld - enum EObjectFlags(ObjectMacros.h)
    
    ```cpp
    /** flags describing an object instance */
    // [ ] see the unreal code
    enum EObjectFlags
    {
        RF_Transactional			=0x00000008,	///< Object is transactional.
        RF_ClassDefaultObject		=0x00000010,	///< This object is used as the default template for all instances of a class. One object is created for each class
    		RF_ArchetypeObject			=0x00000020,	///< This object can be used as a template for instancing objects. This is set on all types of object templates
    };
    ```
    
    </aside>
    

#### // 10 - Foundation - CreateWorld - class ULevel(Level.h)

```cpp
/**
 * the level object:
 * contains the level's actor list, BSP information, and brush list
 * every level has a World as its Outer and can be used as the PersistentLevel, however,
 * when a Level has been streamed in the OwningWorld represents the World that it is a part of 
 */

/**
 * a level is a collection of Actors (lights, volumes, mesh instances etc)
 * multiple levels can be loaded and unloaded into the World to create a streaming experience 
 */

// haker:
// Level?
// - level == collection of actors:
//   - examples of actors:
//     - light, static-mesh, volume, brush(e.g. BSP brush: binary-search-partitioning), ...
//       [ ] explain BSP brush with the editor
// - rest of content will be skipped for now:
//   - when we cover different topics like world-partition, streaming etc, we will visit it again
class ULevel : public UObject
{
		/** array of all actors in this level, used by FActorIteratorBase and derived classes */
		// haker: this is the member variable which contains a list of AActor
		TArray<TObjectPtr<AActor>> Actors;
		
		/** cached level collection that this level is contained in */
		FLevelCollection* CachedLevelCollection;
}
```

#### // 12 - Foundation - CreateWorld - class AActor(Actor.h)

<aside>  
📝

Actor = ActorComponent의 집합

replication의 단위가 됨

#### Actor의 초기화

- 액터는 동적으로 스폰되는 개념이 아닌 월드(Outer)의 서브 오브젝트로 포함된 관계, 그러므로 레벨이 로딩 될 때 이미 액터는 스폰됨

1. PostLoad : 레벨이 스트리밍 되거나 로딩 되었을 때 호출
2. OnComponentCreated : 액터를 구성하는 컴포넌트를 생성
3. PreRegisterAllComponents : 생성한 액터 컴포넌트를 등록 하기 전에 호출
4. RegisterComponent : 생성한 컴포넌트를 등록, 메인 월드(에디터, 게임)이외에 피직스, 랜더 월드에 Actor의 분신(State라고 함)을 추가하는 과정
5. PostResigerAllComponents : 생성한 컴포넌트를 등록한 후에 호출
6. UserConstructionScript : BP의 Hiarachy 창으로 추가한(Simple Construction) 것이 아닌 BP의 ConstructionScript 이벤트를 통해 추가한 컴포넌트들을 생성
7. PreInitializeComponents : 컴포넌트 들을 초기화하기 전
8. InitializeComponent : 컴포넌트 초기화
    1. 여기서 별처리는 일어나지 않고 유저가 오버라이드하여 원하는 코드를 실행 시키는 단계
9. Activate : 액터 활성화, 밑에 주석에는 반대로 되어있는데 InitializeComponent 단계에서 auto 뭐시기 상태일 경우 Activate 호출로 이어짐
    1. 여기서 Tick Function이 등록된다고 함(중요)
10. PostInitializeComponents
11. Beginplay : 본격적으로 Tick이 돌기 직전, 이 이후부터는 Tick이 돌아감  
    </aside>

```cpp
// haker: 
// - Actor:
//   - AActor == collection of ActorComponents (-> Entity-Component structure)
//   - ActorComponent's examples:
//     - UStaticMeshComponent, USkeletalMeshComponent, UAudioComponent, ... etc.
//   - (networking) the unit of replication for propagating updates for state and RPC calls
//
// - Actor's Initializations:
//   1. UObject::PostLoad:
//     - when you place AActor in ULevel, AActor will be stored in ULevel's package
//     - in game build, when ULevel is streaming (or loaded), the placed AActor is loaded and call UObject::PostLoad()
//       - FYI, the UObject::PostLoad() is called when [object need-to-be-spawned]->[asset need-to-be-load]->[after loaded call post-event UObject::PostLoad()]
//   
//   2. UActorComponent::OnComponentCreated:
//     - AActor is already 'spawned'
//     - notification per UActorComponent
//     - one thing to remember is 'component-creation' first!
//   
//   3. AActor::PreRegisterAllComponents:
//     - pre-event for registering UActorComponents to AActor
//     - one thing to remember:
//       - *** [UActorComponent Creation] ---then---> [UActorComponent Registration] ***
//   
//   4. AActor::RegisterComponent:
//     - incremental-registration: register UActorComponent to AActor over multiple frames
//     - what is register stage done?
//       - register UActorComponent to worlds (GameWorld[UWorld], RenderWorld[FScene], PhysicsWorld[FPhysScene], ...)
//       - ***initialize state*** required from each world
//   
//   5. AActor::PostRegisterAllComponents:
//     - post-event for registering UActorComponents to AActor
//
//   6. AActor::UserConstructionScript:
//     [ ] explain UserConstructionScript vs. SimpleConstructionScript in the editor:
//        - UserConstructionScript: called a function for creating UActorComponent in BP event graph
//        - SimpleConstructionScript: construct UActorComponent in BP viewport (hierarchical)
//
//   7. AActor::PreInitializeComponents:
//     - pre-event for initializing UActorComponents of UActor
//     - [UActorComponent Creation]-->[UActorComponent Register]-->[UActorComponent Initialization]
//
//   8. UActorComponent::Activate:
//     - before initializing UActorComponent, UActorComponent's activation is called first
//
//   9. UActorComponent::InitializeComponent:
//     - we can think of UActorComponent's initialization as two stages:
//       1. Activate
//       2. InitializeComponent
//
//  10. AActor::PostInitializeComponents:
//     - post-event when UActorComponents initializations are finished
// 
//  11. AActor::Beginplay:
//     - when level is ticking, AActor calls BeginPlay()
//
//  Diagrams:
// ┌─────────────────────┐                                                                    
// │ UObject::PostLoad() │                                                                    
// └─────────┬───────────┘                                                                    
//           │                                                                                
//           │                                                                                
// ┌─────────▼───────────┐      ┌────────────────────────────────────────────────────────────┐
// │ AActor Spawn        ├─────►│                                                            │
// └─────────┬───────────┘      │  UActorComponent Creation:                                 │
//           │                  │   │                                                        │
//           │                  │   └──UActorComponent::OnComponentCreated()                 │
//           │                  │                                                            │
//           │                  ├────────────────────────────────────────────────────────────┤
//           │                  │                                                            │
//           │                  │  UActorComponent Register:                                 │
//           │                  │   │                                                        │
//           │                  │   ├──AActor::PreRegisterAllComponents()                    │
//           │                  │   │                                                        │
//           │                  │   ├──For each UActorComponent in AActor's UActorComponents │
//           │                  │   │   │                                                    │
//           │                  │   │   └──AActor::RegisterComponent()                       │
//           │                  │   │                                                        │
//           │                  │   └──AActor::PostRegisterAllComponents()                   │
//           │                  │                                                            │
//           │                  ├────────────────────────────────────────────────────────────┤
//           │                  │  AActor::UserConstructionScript()                          │
//           │                  ├────────────────────────────────────────────────────────────┤
//           │                  │                                                            │
//           │                  │  UActorComponent Initialization:                           │
//           │                  │   │                                                        │
//           │                  │   ├──AActor::PreInitializeComponents()                     │
//           │                  │   │                                                        │
//           │                  │   ├──For each UActorComponent in AActor's UActorComponents │
//           │                  │   │   │                                                    │
//           │                  │   │   ├──UActorComponent::Activate()                       │
//           │                  │   │   │                                                    │
//           │                  │   │   └──UActorComponent::InitializeComponent()            │
//           │                  │   │                                                        │
//           │                  │   └──AActor::PostInitializeComponents                      │
//           │                  │                                                            │
//           │                  └────────────────────────────────────────────────────────────┘
//           │                                                                                
//  ┌────────▼───────────┐      ┌────────────────────┐                                        
//  │ AActor Preparation ├─────►│ AActor::BeginPlay()│                                        
//  └────────────────────┘      └────────────────────┘                                        

```

```cpp
class AActor : public UObject
{
		/** all ActorComponents owned by this Actor; stored as a Set as actors may have a large number of components */
		// haker: all components in AActor is stored here
		// 내부적으로 컴포넌트를 분류하는 별도의 컨테이너들도 존재하지만 모든 컴포넌트는 일단 이 안에 들어옴
		TSet<TObjectPtr<UActorComponent>> OwnedComponents;
		
		// 이 밑의 플래그는 어느 단계까지 초기화가 진행됬는지를 판볋하는 변수
		// 1바이트에 1비트만 사용한다는 의미
		// 그럼 저런 변수가 8개가 넘어가면 어떻게 되나?
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
		// 이건 앞에 나온 1바이트 공간을 공유할까?
		EActorBeginPlayState ActorHasBegunPlay : 2;
		
		/** 
		 * the component that defines the transform (location, rotation, scale) of this Actor in the world
		 * all other components must be attached to this one somehow
		 */
		// haker: AActor manages its components(UActorComponent) in a form of scene graph using USceneComponent:
		// [ ] see BP's viewport (refer to SCS[Simple Construction Script])
		//
		// Diagram:                                                                                         
		// AActor                                                                                
		//  │                                                                                    
		//  ├──OwnedComponents: TSet<TObjectPtr<UActorComponent>                                 
		//  │    ┌──────────────────────────────────────────────────────────────────────────┐    
		//  │    │ [Component0, Component1, Component2, Component3, Component4, Component5] │    
		//  │    │                                                                          │    
		//  │    └──────────────────────────────────────────────────────Linear Format───────┘    
		//  └──RootComponent: TObjectPtr<USceneComponent>  
		//     루트 노드                                      
		//       ┌────────────────────────────┐                                                  
		//       │ Component0 [RootComponent] │                                                  
		//       │  │                         │                                                  
		//       │  ├──Component1             │                                                  
		//       │  │   │                     │                                                  
		//       │  │   ├──Component2         │                                                  
		//       │  │   │                     │                                                  
		//       │  │   └──Component3         │                                                  
		//       │  │       │                 │                                                  
		//       │  │       └──Component4     │                                                  
		//       │  │                         │                                                  
		//       │  └──Component5             │                                                  
		//       │                            │                                                  
		//       └────Hierachrical Format─────┘                                                  
		//      
		// 컴포넌트들 사이의 계층 구조(상대 좌표)를 가능케하는 기준                                                                                 
		TObjectPtr<USceneComponent> RootComponent;

		/** The UChildActorComponent that owns this Actor */
		// haker: we are not covering UChildActorComponent in detail, but let's cover concept of UChildActorComponent
		// - UChildActorComponent supports connection between Actor and Actor
		// - we only have Actor-ActorComponent connection, how to support Actor-Actor connection?
		//   - see Diagram:                                                                                               
		//     ┌──────────┐                         ┌──────────┐                           ┌──────────┐              
		//     │  Actor0  ├────────────────────────►│  Actor1  ├──────────────────────────►│  Actor2  │              
		//     └──────────┘                         └──────────┘                           └──────────┘              
		//                                                                                                           
		//     RootComponent                        RootComponent ◄──────────┐             RootComponent             
		//      │                                    │                       │              │                        
		//      ├───Component1◄──────────┐           ├───Component1     Actor2-Actor1       └───Component1           
		//      │                        │           │                       │                                       
		//      ├───Component2    Actor1-Actor0      └───Component2          └─────────────ParentComponent           
		//      │    │                   │                                                                           
		//      │    └───Component3      └──────────ParentComponent                                                  
		//      │                                                                                                    
		//      └───Component4                                                                                       
		//                                                                                                           
		//                                                                                                           
		//                 ┌──────────────────────────────────────┐                                                  
		//                 │                                      │                                                  
		//                 │      Actor0 ◄───                     │                                                  
		//                 │       │                              │                                                  
		//                 │       └─RootComponent                │                                                  
		//                 │          │                           │                                                  
		//                 │          └─Component1                │                                                  
		//                 │             │                        │                                                  
		//                 │             └─Actor1 ◄───            │                                                  
		//                 │                │                     │                                                  
		//                 │                └─RootComponent       │                                                  
		//                 │                   │                  │                                                  
		//                 │                   └─Actor2 ◄────     │                                                  
		//                 │                                      │                                                  
		//                 │                                      │                                                  
		//                 └──────────────────────────────────────┘  
		// 얘는 이름만 보면 페이크에 가까움
		// 액터와 액터 사이에 결합을 위해 사용됨, 액터와 액터는 계층 구조로 묶을 수 없는데 그런 한계를 극복하는 방법
		// (전 프로젝트에서는 세이브 포인트에 StartPoint를 ChildActorComponent로 묶었음, 
		// 근데 그러다보니 초기화가 재대로 이루어지지 않는 문제가 있었음)    
		// 자신을 물고 있는 UChildActorComponent를 weakptr의 형태로 참조?
		TWeakObjectPtr<UChildActorComponent> ParentComponent;
};
```

#### // 17 - Foundation - CreateWorld - class USceneComponent(SceneComponent.h)

USceneComponent는 단순히 좌표를 가지고 있는 컴포넌트가 아니고  
내부에 자신의 부모와 자식에 대한 정보를 가지므로써 계층 구조를 성립시키는 컴포넌트임

```cpp
/**
 * a SceneComponent has a transform and supports attachment, but has no rendering or collision capabilities
 * useful as a 'dummy' component in hierarchy to offset others 
 */
// scene-graph : 부모의 상대 좌표를 가지므로써 트랜스폼을 용이하기 하기 위한 구조
// haker: as we covered, USceneComponent supports scene-graph:
// - what is scene-graph for?
//   - supports hierarichy:
//     - representative example is 'transforms'
class USceneComponent : public UActorComponent
{
    /** get the SceneComponent we are attached to */
    USceneComponent* GetAttachParent() const
    {
        return AttachParent;
    }

    // haker: with AttachParent and AttachChildren, it supports tree-structure for scene-graph

    /** what we are currently attached to. if valid, RelativeLocation etc. are used relative to this object */
    UPROPERTY(ReplicatedUsing=OnRep_AttachParent)
    TObjectPtr<USceneComponent> AttachParent;

    /** list of child SceneComponents that are attached to us. */
    // 왜 얘는 Transient? 직렬화를 안해? 그럼 저장할 때는 자기 부모에 대한 정보만 저장하는건가? 
    UPROPERTY(ReplicatedUsing = OnRep_AttachChildren, Transient)
    TArray<TObjectPtr<USceneComponent>> AttachChildren;
};
```

</aside>