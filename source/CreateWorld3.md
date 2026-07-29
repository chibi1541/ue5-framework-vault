---
summarize: true
---


## Init World

#### // 28 - Foundation - CreateWorld - UWorld::InitWorld(World.cpp)

```cpp
/** initializes the world, associates the persistent level and sets the proper zones */
void InitWorld(const FWorldInitializationValues IVS = FWorldInitializationValues())
{
    // haker: CoreUObjectDelegate has delegates related to GC events -> USEFUL!
    // FCoreUObjectDelegates가 나왔다, 여긴 UObject, GC관련된 유용한 델리게이트들이 많다고 함
    FCoreUObjectDelegates::GetPostGarbageCollect().AddUObject(this, &UWorld::OnPostGC);

    // haker: we initialize UWorldSubsystems
    InitializeSubsystems();
    
    FWorldDelegates::OnPreWorldInitialization.Broadcast(this, IVS);
  
	  // 이미 Persistent Level에 생성한 WorldSettings을 가져옴
    // 여기서 필요한 이유 : PhysicsScene을 구성할 때 필요 셋팅을 AWorldSettings이 가지기 때문
    AWorldSettings* WorldSettings = GetWorldSettings();
    if (IVS.bInitializeScenes)
    {
        if (IVS.bCreatePhysicsScene)
        {
            // haker: we skip the detail of phsics scene
            // - create physics world
            CreatePhysicsScene(WorldSettings);
        }

        bShouldSimulatePhysics = IVS.bShouldSimulatePhysics;

        // haker: create render world (== FScene)
        // - render world is FScene. we'll cover this later 
        bRequiresHitProxies = IVS.bRequiresHitProxies;
        // 랜더링 월드 생성
        GetRendererModule().AllocateScene(this, bRequiresHitProxies, IVS.bCreateFXSystem, GetFeatureLevel());
    }
    
    // Prepare AI systems
		if (WorldSettings)
		{
			if (IVS.bCreateNavigation || IVS.bCreateAISystem)
			{
				if (IVS.bCreateNavigation)
				{
					FNavigationSystem::AddNavigationSystemToWorld(*this, FNavigationSystemRunMode::InvalidMode, WorldSettings->GetNavigationSystemConfig(), /*bInitializeForWorld=*/false);
				}
				if (IVS.bCreateAISystem && WorldSettings->IsAISystemEnabled())
				{
					CreateAISystem();
				}
			}
		}
    
  
    // haker: add persistent level
    // - you can think of persistent level to have world info(AWorldSettings)
    Levels.Empty(1);
    Levels.Add(PersistentLevel);
    
    PersistentLevel->OwningWorld = this;
    PersistentLevel->bIsVisible = true;
  
    // initialize DefaultPhysicsVolume for the world
    // spawned on demand by this function
    
    // Physics Scene이 없어도 PhysicsVolume은 생성해야 함
    // 그렇지 않으면 콜리전 충돌에 대한 판정이 일어나지 않음(추측에 영역)
    DefaultPhysicsVolume = GetDefaultPhysicsVolume();
  
    // find gravity
    if (GetPhysicsScene())
    {
        FVector Gravity = FVector( 0.f, 0.f, GetGravityZ() );
        GetPhysicsScene()->SetUpForFrame( &Gravity, 0, 0, 0, 0, 0, false);
    }

    // create physics collision handler
    if (IVS.bCreatePhysicsScene)
    {
        //...
	  }
	  
    // haker: from here, UWorld's URL is from Persistnet's URL
    URL = PersistentLevel->URL;
#if WITH_EDITORONLY_DATA || 1
    CurrentLevel = PersistentLevel;
#endif

    // haker: make sure all level collection types are instantiated
    ConditionallyCreateDefaultLevelCollections();
  
    // we're initialized now:
    bIsWorldInitialized = true;
    bHasEverBeenInitialized = true;

    FWorldDelegate::OnPostWorldInitialization.Broadcast(this, IVS);

    PersistentLevel->OnLevelLoaded();

    // haker: notify streaming manager that new level is added
    IStreamingManager::Get().AddLevel(PersistentLevel);

    // haker: call PostInitialize() on each UWorldSubsystem
    PostInitializeSubsystems();

    BroadcastLevelsChanged();

    // haker: let's wrap it up what InitWorld() have done:
    // 1. initialize WorldSubsystemCollection
    // 2. allocate(create) render-world(FScene) and physics-world(FPhysScene)
}
```


<aside>  
📝

#### // 29 - Foundation - CreateWorld - UWorld::InitializeSubsystems(World.cpp)

```cpp
/** initialize all world subsystems */
void InitializeSubsystems()
{
    SubsystemCollection.Initialize(this);
}
```

</aside>


#### Foundation - CreateWorld - FSubsytemCollectionBase::Initialize(SubsystemCollection.cpp)

이 로직이 크게 바뀜 World의 Subsystem을 관리하는 방식이 바뀐듯함

FSubsystemCollectionInitialization 이게 subsystem의 초기화를 담당하도록 추가된 듯

```cpp
/** initialize the collection of systems; systems will be created and initialized */
void Initialize(UObject* NewOuter)
{
    // already initialized
    if (Outer)
    {
        return;
    }

    // haker: we set NewOuter as UWorld
    Outer = NewOuter;
    // BaseType은 생성자에서 이미 할당 됨
    
    ****if (ensure(BaseType) && ensure(SubSystemMap.Num() == 0))
    {
	      check(IsInGameThread());

				// 예전에는 UDynamicSubsystem 이랑 Uubsystem를 따로 구분했는데 요즘은 섞어 놓는듯?
        // haker: BaseType is UWorldSubsystem for UWorld::SubsystemCollection
        // - calling GetDerivedClasses() collects all classes derived from UWorldSubsystem
        TArray<UClass*> SubsystemClasses;
        
        // haker: I eliminate UDynamicsSubsystem handling codes for simplicity
        // - the below code is about non-UDynamicSubsystem e.g. UWorldSubsystem
        if (BaseType->IsChildOf(UDynamicSubsystem::StaticClass()))
				{
					for (const TPair<FName, TArray<UClass*>>& ModuleClasses : GlobalDynamicSystemModuleMap)
					{
						for (UClass* SubsystemClass : ModuleClasses.Value)
						{
							if (SubsystemClass->IsChildOf(BaseType))
							{
								SubsystemClasses.Add(SubsystemClass);
							}
						}
					}
				}
				else
				{
					// 여기서 클래스 정보만 수집
          GetDerivedClasses(BaseType, SubsystemClasses, true);
				}
          
				AddAndInitializeSubsystem(SubsystemClass);

        // haker: see the definition of GlobalSubsystemCollections
        // Subsystem의 생성을 글로벌하게 체크
        GlobalSubsystemCollections.Add(this);
    }
}
```

#### Foundation - CreateWorld - FSubsystemCollectionInitialization(SubsystemCollection.h)

```cpp
struct FSubsystemCollectionInitialization
{
	// Classes of subsystem we intend to initialize, for handling calls to GetSubsystem during initialization
	// Key is a base/interface/concrete class which may be used in a call to GetSubsystem
	// Value is the concrete class that will be returned
	// So when key == value, this is a concrete class we intend to initialize
	TMap<UClass*, UClass*> ClassMap;

	// List of classes to be initialized. Classes may be initialized before we reach them in the queue via
	// re-entrancy.
	// We also may add more classes to the queue from re-entrancy, e.g. loading modules.
	TArray<UClass*> Queue;
};
```

#### Foundation - CreateWorld - FSubsytemCollectionBase::AddAndInitializeSubsystem(SubsystemCollection.cpp)

```cpp
void AddAndInitializeSubsystems(UClass* SubsystemClass)
{
	FSubsystemCollectionInitialization LocalInitialization;

	for (UClass* Class : SubsystemClasses)
	{
		// only add instances for non abstract Subsystems
    // haker:
    // - UClass::ClassFlags has class information
	  // 여기서 UWorldSubsystem(Subsystem들이 상속받는 상위 클래스)는 걸러짐
		if (Class->HasAllClassFlags(CLASS_Abstract) || Class->GetAuthoritativeClass() != Class)
		{
			continue;
		}
		
		// haker:
    // - ShouldCreateSubsystem() determines whether we create subsystem or not
    //    - override this method to control the flow of subsystem creation
    // - CDO is sufficient to call ShouldCreateSubsystem()
		if (!Class->GetDefaultObject<USubsystem>()->ShouldCreateSubsystem(Outer))
		{
			UE_LOGFMT(LogSubsystemCollection, Verbose, "Not creating subsystem of class {Class} as it returned false from ShouldCreateSubsystem",
				FTopLevelAssetPath(Class));
			continue;
		}
		LocalInitialization.ClassMap.Add(Class, Class);
		LocalInitialization.Queue.Add(Class);
	}

	// Store a map of classes we intend to initialize for re-entrant calls to GetSubsystem 
	for (UClass* ConcreteClass : SubsystemClasses)
	{
		// Only process classes which were added mapping to themselves above, not more classes which
		// were added during this loop
		if (LocalInitialization.ClassMap.FindRef(ConcreteClass) != ConcreteClass)
		{
			continue;
		}
		for (UClass* Parent = ConcreteClass->GetSuperClass();
			Parent != nullptr && Parent != BaseType && !LocalInitialization.ClassMap.Contains(Parent);
			Parent = Parent->GetSuperClass())
		{
			LocalInitialization.ClassMap.Add(Parent, ConcreteClass);
		}

		for (const FImplementedInterface& Interface : ConcreteClass->Interfaces)
		{
			if (!LocalInitialization.ClassMap.Contains(Interface.Class))
			{
				LocalInitialization.ClassMap.Add(Interface.Class, ConcreteClass);
			}
		}
	}

	// If we're re-entering initialization (e.g. loading a module during subsystem initialization) then add
	// our classes to the existing set/queue and allow the outer loop to do the creation
	if (Initialization)
	{
		for (const TPair<UClass*, UClass*>& Pair : LocalInitialization.ClassMap)
		{
			if (!Initialization->ClassMap.Contains(Pair.Key))
			{
				Initialization->ClassMap.Add(Pair.Key, Pair.Value);	
			}
		}
		Initialization->Queue.Append(LocalInitialization.Queue);
	}
	else 
	{
		// Only mark ourselves as populating now - some subsystems call GetSubsystem in their ShouldCreateSubsystem 
		TGuardValue PopulatingGuard(Initialization, &LocalInitialization);
		for (TPair<UClass*, UClass*> Pair : LocalInitialization.ClassMap)
		{
			// Only initialize the requested classes, other keys are for finding via the inheritence hierarchy 
			// with re-entrant calls to GetSubsystem
			// Skip creation if we created the subsystem re-entrantly
			if (Pair.Key == Pair.Value && !SubsystemMap.Contains(Pair.Key))
			{
				AddAndInitializeValidatedSubsystem(Pair.Key);
			}
		}
	}
}
```


#### Foundation - CreateWorld - FSubsytemCollectionBase::AddAndInitializeValidatedSubsystem(SubsystemCollection.cpp)

```cpp
USubsystem* FSubsystemCollectionBase::AddAndInitializeValidatedSubsystem(UClass* SubsystemClass)
{
  // haker: create new USubsystem by SubsystemClass (in our case, the class is UWorldSubsystem)
  // Subsytem인 이유가 추상적이긴 하지만 이렇게 만들어진 애들의 Outer의 Subobject 개념이기 때문에
  // 그렇기 때문에 상위 outer가 살아 있는한 GC의 대상이 되지 않음
	USubsystem* Subsystem = NewObject<USubsystem>(Outer, SubsystemClass);
	SubsystemMap.Add(SubsystemClass,Subsystem);
	Subsystem->InternalOwningSubsystem = this;
	Subsystem->Initialize(*this);
	
	// Add this new subsystem to any existing maps of base classes to lists of subsystems
	// Not calling FatalErrorIfIteratingSubsystems because adding to the end of the array is safe for index-based iteration
	for (TPair<UClass*, TUniquePtr<FSubsystemArray>>& Pair : SubsystemArrayMap)
	{
		const bool bIsInterface = Pair.Key->IsChildOf<UInterface>();
		if ((!bIsInterface && SubsystemClass->IsChildOf(Pair.Key)) || 
			(bIsInterface && SubsystemClass->ImplementsInterface(Pair.Key)))
		{
			Pair.Value->Subsystems.Add(Subsystem);
		}
	}
}
```

#### // 33 - Foundation - CreateWorld - UWorld::GetWorldSettings(World.cpp)

```cpp
/** returns the AWorldSettings actor associated with this world */
AWorldSettings* GetWorldSettings(bool bCheckStreamingPersistent = false, bool bChecked = true) const
{
    AWorldSettings* WorldSettings = nullptr;
    if (PersistentLevel)
    {
        // haker: do you remember that we create AWorldSettings setting its outer as PersistentLevel?
        WorldSettings = PersistentLevel->GetWorldSettings(bChecked);
        if (bCheckStreamingPersistent)
        {
            //...
        }
    }
    return WorldSettings;
}
```

#### // 34 - Foundation - CreateWorld - UWorld::GetDefaultPhysicsVolume(World.h)

```cpp
/** returns the default physics volume and creates it if necessary */
APhysicsVolume* GetDefaultPhysicsVolume() const { return DefaultPhysicsVolume ? ToRawPtr(DefaultPhysicsVolume) : InternalGetDefaultPhysicsVolume(); }

APhysicsVolume* InternalGetDefaultPhysicsVolume() const
{
    // haker: if we don't have any DefaultPhysicsVolume yet, create new one
    if (DefaultPhysicsVolume == nullptr)
    {
        // haker: we get the PhysicssVolume's class to instantiate:
        // - WorldSettings have class definitions which are needed for the world, which could change overall behavior for world
        // - you could also override the class in WorldSettings
        AWorldSettings* WorldSettings = GetWorldSettings(false, false);
        UClass* DefaultPhysicsVolumeClass = (WorldSettings ? WorldSettings->DefaultPhysicsVolumeClass : nullptr);

        if (DefaultPhysicsVolumeClass == nullptr)
        {
            DefaultPhysicsVolumeClass = ADefaultPhysicsVolume::StaticClass();
        }

        // spawn volume:
        // haker: here is FActorSpawnParameters
        FActorSpawnParameters SpawnParams;
        SpawnParams.bAllowDuringConstructionScript = true;
        {
            UWorld* MutableThis = const_cast<UWorld*>(this);
            MutableThis->DefaultPhysicsVolume = MutableThis->SpawnActor<APhysicsVolume>(DefaultPhysicsVolumeClass, SpawnParams);
            MutableThis->DefaultPhysicsVolume->Priority = -1000000;
        }
    }
    return DefaultPhysicsVolume;
}
```

#### // 36 - Foundation - CreateWorld - UWorld::ConditionallyCreateDefaultLevelCollections(World.cpp)

```cpp
/** creates the dynamic source and static level collections if they don't already exists */
void ConditionallyCreateDefaultLevelCollections()
{
    LevelCollections.Reserve((int32)ELevelCollectionType::MAX);

    // create main level collection; the persistent level will always be considered dynamic
    if (!FindCollectionByType(ELevelCollectionType::DynamicSourceLevels))
    {
        // default to the dynamic source collection
        // Persistent Level은 dynamic source collection?
        ActiveLevelCollectionIndex = FindOrAddCollectionByType_Index(ELevelCollectionType::DynamicSourceLevels);

        // haker: dynamc/static level collections are set its persistent level for World's persistent level
        // 항상 World에서 생성한 Persistent Level을 설정함
        LevelCollections[ActiveLevelCollectionIndex].SetPersistentLevel(PersistentLevel);

        // don't add the persistent level if it is already a member of another collection
        // this may be the case if, for example, this world is the outer of a streaming level,
        // in which case the persistent level may be in one of the collections in the streaming level's OwningWorld
        if (PersistentLevel->GetCachedLevelCollection() == nullptr)
        {
            // haker: if persistent level is not set, defaultly add it to the dynamic level collection
            LevelCollections[ActiveLevelCollectionIndex].AddLevel(PersistentLevel);
        }
    }

    if (!FindCollectionByType(ELevelCollectionType::StaticLevels))
    {
        FLevelCollection& StaticCollection = FindOrAddCollectionByType(ELevelCollectionType::StaticLevels);
        StaticCollection.SetPersistentLevel(PersistentLevel);
    }
}
```

#### Foundation - CreateWorld - UWorld::FindOrAddCollectionByType_Index(World.cpp)

```cpp
int32 FindOrAddCollectionByType_Index(const ELevelCollectionType InType)
{
    const int32 FoundIndex = FindCollectionIndexByType(InType);

    if (FoundIndex != INDEX_NONE)
    {
        return FoundIndex;
    }

    // Not found, add a new one.
    FLevelCollection NewLC;
    NewLC.SetType(InType);
    return LevelCollections.Add(MoveTemp(NewLC));
}
```

#### Foundation - CreateWorld - UWorld::FindCollectionIndexByType(World.cpp)

```cpp
FLevelCollection* FindCollectionByType(const ELevelCollectionType InType)
{
    for (FLevelCollection& LC : LevelCollections)
    {
        if (LC.GetType() == InType)
        {
            return &LC;
        }
    }
    return nullptr;
}
```


#### // 37 - Foundation - CreateWorld - UWorld::PostInitializeSubsystems(World.cpp)

```cpp
/** finalize initialization of all world subsystems */
void PostInitializeSubsystems()
{
		// 원래는 배열을 받아서 for문을 돌았는데 로직이 변경됨
		// 이것도 월드 내부의 subsystem을 관리하는 방식이 변경되어서 그런듯
		SubsystemCollection.ForEachSubsystem([](UWorldSubsystem* WorldSubsystem)
		{
				WorldSubsystem->PostInitialize();
				WorldSubsystem->EnsureHasCalledPostInitialize();
		});
}
```

#### Foundation - CreateWorld - FObjectSubsystemCollection::ForEachSubsystem(SubsystemCollection.h)

```cpp
/** Perform an operation on all subsystems in the collection */
void ForEachSubsystem(TFunctionRef<void(TBaseType*)> Operation, const TSubclassOf<TBaseType>& SubsystemClass = {}) const
{
	ForEachSubsystemOfClass(SubsystemClass, [Operation=MoveTemp(Operation)](USubsystem* Subsystem){
		Operation(CastChecked<TBaseType>(Subsystem));
	});
}
```

#### Foundation - CreateWorld - FSubsystemCollectionBase::ForEachSubsystemOfClass(SubsystemCollection.cpp)

```cpp
void FSubsystemCollectionBase::ForEachSubsystemOfClass(UClass* SubsystemClass, TFunctionRef<void(USubsystem*)> Operation) const
{
	if (SubsystemClass == nullptr)
	{
		SubsystemClass = USubsystem::StaticClass();
	}

	if (const FSubsystemArray* List = FindAndPopulateSubsystemArray(SubsystemClass))
	{
		// iteration으로 도는 동안에 안에 있는 값 지우지 않도록 하기 위한 세이프 가드?
		TGuardValue<bool> IterationGuard{List->bIsIterating, true};
		for (int32 i=0; i < List->Subsystems.Num(); ++i)
		{
			Operation(List->Subsystems[i]);
		}
	}
}
```

#### Foundation - CreateWorld - FSubsystemCollectionBase::FindAndPopulateSubsystemArray(SubsystemCollection.cpp)

```cpp
const FSubsystemCollectionBase::FSubsystemArray* FSubsystemCollectionBase::FindAndPopulateSubsystemArray(UClass* SubsystemClass) const
{
	// NOTE: There is no thread safety here for multiple threads trying to access a subsystem by a base class/interface.
	// Ideally there should be.
	const bool bIsInterface = SubsystemClass->IsChildOf<UInterface>();
	if (!SubsystemArrayMap.Contains(SubsystemClass))
	{
		FSubsystemArray NewList;
		for (auto Iter = SubsystemMap.CreateConstIterator(); Iter; ++Iter)
		{
			UClass* KeyClass = Iter.Key();
			if ((!bIsInterface && KeyClass->IsChildOf(SubsystemClass)) || 
				(bIsInterface && KeyClass->ImplementsInterface(SubsystemClass)))
			{
				NewList.Subsystems.Add(Iter.Value());
			}
		}
		if (NewList.Subsystems.Num())
		{
			return SubsystemArrayMap.Add(SubsystemClass, MakeUnique<FSubsystemArray>(MoveTemp(NewList))).Get();
		}
		// If nothing was found, we don't want to store this key in the map as we remove lists when they become empty.
		// We also don't pass the keys in SubsystemArrayMap to GC.
		return nullptr;
	}

	const TUniquePtr<FSubsystemArray>& List = SubsystemArrayMap.FindChecked(SubsystemClass);
	return List.Get();
}
```