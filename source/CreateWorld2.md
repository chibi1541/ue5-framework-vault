---
summarize: true
---


## CreateWorld

#### // 11 - Foundation - CreateWorld - class ULevel(Level.h)

```cpp
class ULevel : public UObject
{
		// ...

		// 이게 월드가 아닌 레벨이 액터를 들고 관리하는 이유?
		/** cached level collection that this level is contained in */
		FLevelCollection* CachedLevelCollection;
		
		// 레벨을 스트리밍 할 때 증분적으로 처리하기 위해 나눈 구간 타입
		enum class EIncrementalComponentState : uint8
		{
		    Init,
		    RegisterInitialComponents,
		#if WITH_EDITOR || 1
				// 게임에서는 이미 쿠킹된 형태이므로 이 단계가 불릴 필요가 없어서 에디터 온리
		    RunConstructionScripts,
		#endif
		    Finalize,
		};
		
		/** the current stage for incrementally updating actor components in the level */
		// haker: we already covered AActor's initialization steps
		EIncrementalComponentState IncrementalComponentState;
		
		// 여기도 각 단계를 체크하기 위한 비트 플래그
		/** whether the actor referenced by CurrentActorIndexForUpdateComponents has called PreRegisterAllComponents */
		uint8 bHasCurrentActorCalledPreRegister : 1;
		
		/** whether components are currently registered or not */
		uint8 bAreComponentsCurrentlyRegistered : 1;
		
		// RegisterInitialComponents 단계 내부에서 증분적으로 진행된 단계를 체크하는 변수
		/** current index into actors array for updating components */
		// haker: tracking actor index in ULevel's ActorList to support incremental update
		int32 CurrentActorIndexForIncrementalUpdate;
		
		// 이건 나중에 Tick에서 한번에 확인...
		/** data structures for holding the tick functions */
		// haker: for now, member variables related to tick function are skipped
		FTickTaskLevel* TickTaskLevel;
}
```

#### // 19 - Foundation - CreateWorld - struct FLevelCollection(World.h)

```cpp
// haker: FLevelCollection is collection based on ELevelCollectionType
struct FLevelCollection
{
    /** the type of this collection */
    ELevelCollectionType CollectionType;

    /**
     * the persistent level associated with this collection
     * the source collection and the duplicated collection will have their own instances 
     */
    // haker: usually OwnerWorld's PersistentLevel
    TObjectPtr<class ULevel> PersistentLevel;

    /** all the levels in this collection */
    TSet<TObjectPtr<ULevel>> Levels;
    
    // 왜 outer라는 개념이 있는데도 굳이 OwningWorld를 따로 관리해야하는가?
    // 레벨 컴포지션에서는 레벨의 outer가 반드시 owningworld가 아니라고 함?
    // 이 부분은 디버깅이 필요할 듯
    /**
     * the world that has this level in its Levels array
     * this is not the same as GetOuter(), because GetOuter() for a streaming level is a vestigial world that is not used
     * it should not be accessed during BeginDestroy(), just like any other UObject references, since GC may occur in any order
     */
    // haker: let's understand OwningWorld vs. OuterPrivate
    // - note that my explanation is based on WorldComposition's level streaming or LevelBlueprint's level load/unload manipulation
    //   - World Partition has different concept which usually OwningWorld and OuterPrivate is same
    // - Diagram:                                                                                                        
    //      World0(OwningWorld)──[OuterPrivate]──►Package0(World.umap)                                          
    //       ▲                                                                                                  
    //       │                                                                                                  
    // [OuterPrivate]                                                                                           
    //       │                                                                                                  
    //       │                                                                                                  
    //      Level0(PersistentLevel)-> persistent 레벨은 월드와 1:1                                                                            
    //       │                                                                                                  
    //       │                                                                                                  
    //       ├────Level1──[OuterPrivate]──►World1(왜 여기 월드가 또 있나?, AI에 물어보면 이런거 없다는데?)───[OuterPrivate]───►Package1(Level1.umap)
    //       │    (OwningWorld는 World0)                  
    //       │                                                                                                  
    //       └────Level2───────►World2───────►Package2(Level2.umap) 
    //            (여기도 OwningWorld는 World0)                                              
    TObjectPtr<UWorld> OwningWorld;
};
```

#### // 20 - Foundation - CreateWorld - enum ELevelCollectionType(EngineTypes.h)

```cpp
/** indicates the type of a level collection, used in FLevelCollection */
// haker: don't sticking to its types, we are going to understand it simply like dynamic level vs. static level
// 레벨을 분류할 때 제한 요소를 추가해서 최적화 여지를 두기 위한 타입
enum class ELevelCollectionType : uint8
{
		// DynamicSourceLevels, DynamicDuplicatedLevels를 묶어서 DynamicLevel
    /**
     * the dynamic levels that are used for normal gameplay and the source for any duplicated collections
     * will contain a world's persistent level and any streaming levels that contain dynamic or replicated gameplay actors
     */
    DynamicSourceLevels,

    /** gameplay relevant levels that have been duplicated from DynamicSourceLevels if requested by the game */
    DynamicDuplicatedLevels,

    /**
     * these levels are shared between the source levels and the duplicated levels, and should contain
     * only static geometry and other visuals that are not replicated or affected by gameplay
     * thsese will not be duplicated in order to save memory 
     */
    StaticLevels,

    MAX
};
```

#### // 4 - Foundation - CreateWorld - class UWorld(World.h)

```cpp
class UWorld final
{
	/** array of level collections currently in this world */
	// haker: UWorld has the classified collections of level we have covered in ULevel
	TArray<FLevelCollection> LevelCollections;
	
	// 월드 컬렉션을 TickFunction 기준으로 나눌 때 사용. 나중에...
	/** index of the level collection that's currently ticking */
	int32 ActiveLevelCollectionIndex;
	
	// 월드 생성 시, 피직스 월드 생성 -> 이걸 특정 영역에만 적용하고 싶을 경우(얘를 들어 배경에는 굳이 물리 처리가 필요 없을 때)
	// 이 범위를 조정하는 것이 피직스 볼륨
	/** DefaultPhysicsVolume used for whole game */
	// haker: you can think of physics volume as 3d range of physics engine works (== physics world covers)
	TObjectPtr<APhysicsVolume> DefaultPhysicsVolume;
	
	// 그냥 여기 피직스 Scene이 있다는 정도만 확인
	/** physics scene for this world */
	FPhysScene* PhysicsScene;
	
	// FWorldSubsystemCollection == FObjectSubsystemCollection<UWorldSubsystem> 
	FWorldSubsystemCollection SubsystemCollection;
	
	
	/** line batchers: */
	// ULineBatchComponent는 PrimitiveComponent 즉 SceneComponent로서 실제 좌표를 가지고
	// 랜더되는 아이라서 Actor 계열에 있어야 하는데 이거 왜 UWorld가 가지고 잇는가?
	// 디버깅 기능으로 추가한 녀석들인데 이렇게 되면 월드의 서브오브젝트로 끼어들어가는 형태라
	// 기본 월드 - 레벨 - 액터 - 컴포넌트 구조와는 어긋남
	// 이런 경우 월드가 초기화 할 때 별도로 초기화 과정을 거침
	// haker: debug lines
	// - ULineBatchComponents are resided in UWorld's subobjects
	// UE_DEPRECATED(5.6, "얘들 다 UWorld::GetLineBatcher(ELineBatcherType Type)로 대체됨")
	// 얘들이 월드 Init하는 단계에서 곧바로 컴포넌트 Register를 수행하기 때문에 조금 특별한 취급
	TObjectPtr<class ULineBatchComponent> LineBatcher;
	TObjectPtr<class ULineBatchComponent> PersistentLineBatcher;
	TObjectPtr<class ULineBatchComponent> ForegroundLineBatcher;
}
```

#### // 21 - Foundation - CreateWorld - class USubsystem(Subsystem.h)


특정 Object에 따라서 라이프 사이클을 조절할 수 있는 싱글톤 매니저

꼭 필요한 가상 함수만 재대로 설정해 주면 나머지는 편하게 관리해 줌

```cpp
/**
 * subsystems are auto instanced classes that share the lifetime of certain engine constructs
 * 
 * currently supported subsystem lifetimes are:
 *  Engine		 -> inherit UEngineSubsystem
 *	Editor		 -> inherit UEditorSubsystem
 *	GameInstance -> inherit UGameInstanceSubsystem
 *	World		 -> inherit UWorldSubsystem
 *	LocalPlayer	 -> inherit ULocalPlayerSubsystem
 * 
 * normal example:
 *  class UMySystem : public UGameInstanceSubsystem
 * which can be accessed by:
 *  UGameInstance* GameInstance = ...;
 *  UMySystem* MySystem = GameInstance->GetSubsystem<UMySystem>();
 * 
 * or the following if you need protection from a null GameInstance
 *  UGameInstance* GameInstance = ...;
 *  UMyGameSubsystem* MySubsystem = UGameInstance::GetSubsystem<MyGameSubsystem>(GameInstance);
 * 
 *	You can get also define interfaces that can have multiple implementations.
 *	Interface Example :
 *      MySystemInterface
 *    With 2 concrete derivative classes:
 *      MyA : public MySystemInterface
 *      MyB : public MySystemInterface
 *
 *	Which can be accessed by:
 *		UGameInstance* GameInstance = ...;
 *		const TArray<UMyGameSubsystem*>& MySubsystems = GameInstance->GetSubsystemArray<MyGameSubsystem>();
 */
// haker:
// - USubsystem is the system follows life-time (create/destroy) of a component of the unreal engine:
// - types of subsystems:
//   1. Engine        -> inherit UEngineSubsystem
//   2. Editor        -> inherit UEditorSubsystem
//   3. GameInstance  -> inherit UGameInstanceSubsystem
//   4. World         -> inherit UWorldSubsystem
//   5. LocalPlayer   -> inherit ULocalPlayerSubsystem
// - **the care of lifetime with engine's component like UWorld** is cumbersome:
//   - if you try to use Subsytem, it will be very easy and handy!
// - read examples of how to access subsystems
```

</aside>

```cpp
class USubsystem : public UObject
{
    /**
     * override to control if the Subsystem should be created
     * for example you could only have your system created on servers
     * it is important to note that if using this is becomes very important to null check whenever getting the Subsystem
     * 
     * NOTE: this function is called on the CDO prior to instances being created!!! 
     */
    // UWorldSubsystem : Outer == UWorld
    // UGameInstanceSubsystem : Outer == UGameInstance
    // haker: as the comment describes, this member function is called via CDO(ClassDefaultObject)
    virtual bool ShouldCreateSubsystem(UObject* Outer) const { return true; }

    /** implement this for init/deinit of instances of the system */
    virtual void Initialize(FSubsystemCollectionBase& Collection) {}
    virtual void Deinitialize() {}

    // haker: we are interested UWorld's [FObjectSubsystemCollection<UWorldSubsystem>]
    // - each subsystem has its owner like this
    FSubsystemCollectionBase* InternalOwningSubsystem;
};
```

#### // 23 - Foundation - CreateWorld - class FSubsystemCollectionBase(SubsystemCollection.h)

```cpp
class FSubsystemCollectionBase
{
    // haker:
    // if we are dealing with UWorldSusystem:
    // - Outer is UWorld
    // - BaseType is UWorldSubsystem
    UObject* Outer;
    // 추후에 서브시스템의 인스턴스를 생성할 때 NewObject에 인자로 전달되는 ClassType
    UClass* BaseType;

		// 여기서 하나의 타입만 들어가도록 체크하니까
		// 오직 하니의 인스턴스만 생성 됨
    // haker: mapper from UClass(Subsystem's UClass) to UObject(Subsystem's instance)
    // - at initialization time of FObjectSubsystemCollection<SubsystemType>, one-to-one mappings are supported 
    TMap<TObjectPtr<UClass>, TObjectPtr<USubsystem>> SubsystemMap;

		// UDynamicSubsystem : 같은 Subsystem을 동적으로 하나 더 만들 수 있도록 하는 타입의 Subsystem
		// 존재 이유는? 모르겠당
    // haker: from my guess, to support UDynamicSubsystem, it persist multiple instances for each USubsystem's class type
    mutable TMap<UClass*, TArray<USubsystem*>> SubsystemArrayMap;
}
```

</aside>

<aside>  
📝

## World 구조 정리

AActor와 UActorComponent는 그자체로 계층적 구조를 가지고

이와는 별도로 UObject와 UObject는 OuterPrivate ↔ Subobject 관계로 계층 구조를 형성할 수 있음(UWorld ↔ ULevel ↔ AActor)

Level을 에디터에서 계층적으로 배치할 순 있지만 그게 실제 계층 구조가 아님(레벨은 자신만의 절대 좌표를 가짐), Persistent Level과 Sub Level만 계층적 구조가 성립함

```cpp
    // haker: let's wrap up what we have looked through classes:
    //                                                                                ┌───WorldSubsystem0       
    //                                                        ┌────────────────────┐  │                         
    //                                                 World──┤SubsystemCollections├──┼───WorldSubsystem1       
    //                                                   │    └────────────────────┘  │                         
    //                                                   │                            └───WorldSubsystem2       
    //             ┌─────────────────────────────────────┴────┐                                                 
    //             │                                          │                                                 
    //           Level0                                     Level1                                              
    //             │                                          │                                                 
    //         ┌───┴────┐                                 ┌───┴────┐                                            
    //         │ Actor0 ├────Component0(RootComponent)    │ Actor0 ├─────Component0(RootComponent)              
    //         ├────────┤     │                           ├────────┤      │                                     
    //         │ Actor1 │     ├─Component1                │ Actor1 │      │   ┌──────┐                          
    //         ├────────┤     │                           ├────────┤      └───┤Actor2├──RootComponent           
    //         │ Actor2 │     └─Component2                │ Actor2 │          └──────┘   │                      
    //         ├────────┤                                 ├────────┤                     ├──Component0          
    //         │ Actor3 │                                 │ Actor3 │                     │                      
    //         └────────┘                                 └────────┘                     ├──Component1          
    //                                                                                   │   │                  
    //                                                                                   │   └──Component2      
    //                                                                                   │                      
    //                                                                                   └──Component3         
```

</aside>

#### // 1 - Foundation - CreateWorld - UWorld::CreateWorld(World.cpp)

월드 내부의 구조와 이들의 생성 및 초기화를 살펴봤으니 다시 CreateWorld함수로

```cpp
UWorld* UWorld::CreateWorld(const EWorldType::Type InWorldType, bool bInformEngineOfWorld, FName WorldName, UPackage* InWorldPackage, bool bAddToRoot, ERHIFeatureLevel::Type InFeatureLevel, const InitializationValues* InIVS, bool bInSkipInitWorld)
{
	if (InFeatureLevel >= ERHIFeatureLevel::Num)
	{
		InFeatureLevel = GMaxRHIFeatureLevel;
	}

	UPackage* WorldPackage = InWorldPackage;
	if ( !WorldPackage )
	{
		WorldPackage = CreatePackage(nullptr);
	}

	if (InWorldType == EWorldType::PIE)
	{
		WorldPackage->SetPackageFlags(PKG_PlayInEditor);
	}

	// Mark the package as containing a world.  This has to happen here rather than at serialization time,
	// so that e.g. the referenced assets browser will work correctly.
	if ( WorldPackage != GetTransientPackage() )
	{
		WorldPackage->ThisContainsMap();
	}

	// Create new UWorld, ULevel and UModel.
	const FString WorldNameString = (WorldName != NAME_None) ? WorldName.ToString() : TEXT("Untitled");
	UWorld* NewWorld = NewObject<UWorld>(WorldPackage, *WorldNameString);

	// haker: UObject::SetFlags -> set unreal object's attribute with flag by bit operator (AND(&), OR(|), SHIFT(>>, <<) etc...)
	// - refer to EObjectFlags and UObject::ObjectFlags
	// RF_Transactional 이게 무슨 의미인가? redo, undo 기능을 적용하는 기준
	NewWorld->SetFlags(RF_Transactional);
	NewWorld->WorldType = InWorldType;
	// 이건 랜더링 API 설정, 이게 따로 있는 이유는 디바이스 마다 API가 다르기 때문
	// XBox -> DX, PS5 -> FB뭐시기, Android -> Vulcan
	NewWorld->SetFeatureLevel(InFeatureLevel);
	
	// 여기서 드디어 초기화
	NewWorld->InitializeNewWorld(
	      InIVS ? *InIVS : UWorld::InitializationValues()
	          // haker: as we saw FWorldInitializationValues, the below member functions mark the flag to refer when we create new world
	          .CreatePhysicsScene(InWorldType != EWorldType::Inactive)
	          .ShouldSimulatePhysics(false)
	          .EnableTraceCollision(true)
	          .CreateNavigation(InWorldType == EWorldType::Editor)
	          .CreateAISystem(InWorldType == EWorldType::Editor)
	      , bInSkipInitWorld);
}
```

#### // 25 - Foundation - CreateWorld - UWorld::InitializeNewWorld(World.cpp)

```cpp
void InitializeNewWorld(const InitializationValues IVS = InitializationValues(), bool bInSkipInitWorld = false)
{
    if (!IVS.bTransactional)
    {
        // haker: clear ObjectFlags in UObjectBase (bit-wise operation)
        // 요건 내부를 한번 따라 쳐보자(선생님이 C++ 플머라면 이건 자유자재로 해야 한다 하심)
        // 내부적으로는 OldFlag & ~FlagToClear하는 방식으로 진행
        ClearFlags(RF_Transactional);
    }
  
    // haker: create default persistent level for new world
    // NewObject로 객체 생성 시에 반드시 Outer를 넣어주어야 함
    // UWorld랑 PersistentLevel은 1:1 관계
    // 월드가 바뀌지 않는 이상 내려가지 않는, 스트리밍 되지 않는 레벨이 PersistentLevel
    // 그렇다는건 UWorld와 PersistentLevel은 같은 UPackage 안에 직렬화 된다는 의미
    // Sub Level들은 별도의 UPackage에 묶임 다만 World Partition은 Sub Level이 존재하지 않기 때문에 이런 방식과는 다르다고 함
    PersistentLevel = NewObject<ULevel>(/*Outer*/this, TEXT("PersistentLevel"));
    PersistentLevel->OwningWorld = this;
  
	  // 시차* : 새로 추가한 구문
	  // UModel이 뭐지? 
	  // UModel : BSP 브러쉬의 지오메트리 정보
	  PersistentLevel->Initialize(FURL(nullptr));
		PersistentLevel->Model = NewObject<UModel>(PersistentLevel);
		PersistentLevel->Model->Initialize(nullptr, 1);
  
    // create the WorldInfo actor
    // haker: we are NOT looking into AWorldSettings in detail, but try to understand it with the example
    // [ ] explain AWorldSettings in the editor
    // 에디터에서 보이는 그 월드 셋팅임, 그걸 액터로 만들어서 스폰함
    AWorldSettings* WorldSettings = nullptr;
	  {
			  // 액터를 스폰할 때 사용하는 파라미터
        // haker: when you spawn an Actor, you need to pass FActorSpawnParameters
        FActorSpawnParameters SpawnInfo;
        SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    
        // set constant name for WorldSettings to make a network replication work between new worlds on host and client
        // haker: you can override WorldSettings' class using UEngine's WorldSettingClass
        SpawnInfo.Name = GEngine->WorldSettingsClass->GetFName();

        // haker: we'll cover SpawnActor in the future (I just leave it in to-do list)
        // 현재 월드의 Persistent Level에 종속된 형태로 스폰
        WorldSettings = SpawnActor<AWorldSettings>(GEngine->WorldSettingsClass, SpawnInfo);
    }
    
    // allow the world creator to override the default game mode in case they do not plan to load a level
    if (IVS.DefaultGameMode)
    {
        WorldSettings->DefaultGameMode = IVS.DefaultGameMode;
    }
    
    // haker: persistent level creates AWorldSettings which contain world info like GameMode
	  PersistentLevel->SetWorldSettings(WorldSettings);
	  
#if WITH_EDITOR
    WorldSettings->SetIsTemporaryHiddenInEditor(true);

    if (IVS.bCreateWorldParitition)
    {
        // haker: we skip world partition for now
        // - BUT, lyra is based on World-Partition
    }
#endif

    if (!bInSkipInitWorld)
    {
        // initialize the world
        InitWorld(IVS);

        // update components
        const bool bRerunConstructionScripts = !FPlatformProperties::RequiresCookedData();
        UpdateWorldComponents(bRerunConstructionScripts, false);
    }
}
```

#### // 26 - Foundation - CreateWorld - struct FActorSpawnParameters(World.h)

여기서 주요하게 볼 건 Name이랑 SpawnCollisionHandlingOverride

```cpp
/** struct of optional parameters passed to SpawnActor function(s) */
// haker: I reduce its member variables drastically
struct FActorSpawnParameters
{
		// FName 유니크한 값 -> 내부는 NameEntryIndex = WorldSettings, Number = 0의 형태로 나누어서 보관
		// NameEntryIndex를 별도의 메모리로 항상 올려놓고 중복을 체크함
    /** a name to assign as the Name of the Actor being spawned; if no value is specified, the name of the spawned Actor will be automatically generated using the form [Class]_[Number] */
    // haker: default format of FName is [ClassName]_[Number]:
    // - e.g. WorldSettings_0
    FName Name;

    /** method for resolving collisions at the spawn point; undefined means no override, use the actor's setting */
    ESpawnActorCollisionHandlingMethod SpawnCollisionHandlingOverride;

    /** determines whether or not the actor may be spawned when running a construction script; if true spawning will fail if a construction script is being run */
    uint8 bAllowDuringConstructionScript : 1;
};

/** defines available strategies for handling the case where an actor is spawned in such a way that it penetrates blocking collisions */
// haker: spawning policy depending on collision between object and obstacles
enum class ESpawnActorCollisionHandlingMethod : uint8
{
    /** fall back to default settings */
    Undefined,
    // 충돌 신경 쓰지 말고 무조건 스폰
    /** actor will spawn in desired location, regardless of collisions */
    AlawysSpawn,
    // 스폰 시에 이미 충돌할 것 같은 물체가 있는 경우 스폰하지 않음
    /** actor will try to find a nearby non-colliding location (based on shape components) but will always spawn even if one cannot be found */
    // haker: spawn but adjust a little bit and spawn non-collising location
    AdjustIfPossibleButAlwaysSpawn,
    //...
};
```

#### // Foundation - CreateWorld - enum ESpawnActorCollisionHandlingMethod (EngineTypes.h)

```jsx
/** Defines available strategies for handling the case where an actor is spawned in such a way that it penetrates blocking collision. */
UENUM(BlueprintType)
enum class ESpawnActorCollisionHandlingMethod : uint8
{
	/** Fall back to default settings. */
	Undefined								UMETA(DisplayName = "Default"),
	/** Actor will spawn in desired location, regardless of collisions. */
	AlwaysSpawn								UMETA(DisplayName = "Always Spawn, Ignore Collisions"),
	/** Actor will try to find a nearby non-colliding location (based on shape components), but will always spawn even if one cannot be found. */
	AdjustIfPossibleButAlwaysSpawn			UMETA(DisplayName = "Try To Adjust Location, But Always Spawn"),
	/** Actor will try to find a nearby non-colliding location (based on shape components), but will NOT spawn unless one is found. */
	AdjustIfPossibleButDontSpawnIfColliding	UMETA(DisplayName = "Try To Adjust Location, Don't Spawn If Still Colliding"),
	/** Actor will fail to spawn. */
	DontSpawnIfColliding					UMETA(DisplayName = "Do Not Spawn"),
};
```
