---
summarize: true
---

#### Foundation - CreateWorld - UWorld::UpdateWorldComponents(World.cpp)

```cpp
void UpdateWorldComponents(bool bRerunConstructionScripts, bool bCurrentLevelOnly, FRegisterComponentContext* Context = nullptr)
{
    if (!IsRunningDedicatedServer())
    {
				// 월드에 직접 속해있는 ActorComponent(LineBatcher같은 애들)를 생성 및 등록
    }

    {
        for (int32 LevelIndex = 0; LevelIndex < Levels.Num(); ++LevelIndex)
        {
            ULevel* Level = Levels[LevelIndex];
            // 이 내부는 스트리밍 된 서브 레벨을 구분하는 과정이지만
            // 복잡하기 때문에 일단 여기서는 모든 서브 레벨로 아래의 로직을 수행한다 정도로 인식
            ULevelStreaming* StreamingLevel = FLevelUtils::FindStreamingLevel(Level);
            
            // update the level only if it is visible (or not a streamed level)
            if (!StreamingLevel || Level->bIsVisible)
            {
		            // 위에서 지금까지 봤던 상황은 월드가 직접 ActorComponent를 갖는다는 사실상 일반적이지 않은 상황
                // Actor는 Level에 종속되어 있는게 일반적이기 때문에 아래 상황이 일반적인 초기화
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


#### Foundation - CreateWorld - ULevel::UpdateLevelComponents(Level.cpp)

```cpp
/** update all components of actors associated with this level (aka in Actors array) and creates the BSP model components */
void UpdateLevelComponents(bool bRerunConstructionScripts, FRegisterComponentContext* Context = nullptr)
{
    // update all components in one swoop
    // 첫번째 인자에 0을 넣으면 증분적으로 Update하는 것이 아닌 한번에 싹다 업데이트 해버림
    // bRerunConstructionScripts : BP 편집 후 컴파일 버튼으로 변경 내용을 기반으로 다시 CS를 호출하는 옵션
    // 이미 쿠킹을 진행해서 asset을 최적화(빌드)한 상태라면 해당 옵션은 false?
    IncrementalUpdateComponents(0, bRerunConstructionScripts, Context);
}
```

#### Foundation - CreateWorld - ULevel::IncrementalUpdateComponents(Level.cpp)

NumComponentsToUpdate = 0 인 상황이므로 증분적으로 진행되는게 아닌 레벨의 전 액터 및 액터 컴포넌트가 한번에 Register됨(전 과정 중 증분적으로 진행되는건 RegisterInitialComponents 과정과 RunConstructionScripts만)

```cpp
/** incrementally update all components of actor associated with this level */
void IncrementalUpdateComponents(int32 NumComponentsToUpdate, bool bRerunConstructionScripts, FRegisterComponentContext* Context = nullptr)
{
    // a value of 0 means that we want to update all components
    if (NumComponentsToUpdate != 0)
    {
        // only the game can use incremental update functionality
        // haker: the target for debugging is EditorWorld, so we consider NumComponentsToUpdate is 0
        check(OwningWorld->IsGameWorld());
    }

    // haker: if we pass NumComponentsToUpdate as 0, it will update all components in the level:
    // - this is our case!
    bool bFullyUpdateComponents = (NumComponentsToUpdate == 0);

    // the editor is never allowed to incrementally update components; make sure to pass in a value of zero for NumActorsToUpdate
    // haker: you should understand the below check() function conveniently
    check(bFullyUpdateComponents || OwningWorld->IsGameWorld());

    do
    {
        // haker: see IncrementalComponentState:
        // the incremental update is happened:            
        // - for Init and Finalize is done all at once
        // - in RegisterInitialComponent is done incrementally based on NumComponentsToUpdate
        //
        // ┌──────────────────────────────────┐                       
        // │ EIncrementalComponentState::Init │                       
        // └─┬─┬──────────────────────────────┘                       
        //   │ │                                                      
        //   │ └──SortActorHierarchy()                                
        //   │                                                        
        //   │                                                        
        // ┌─▼─────────────────────────────────────────────────────┐  
        // │ EIncrementalComponentState::RegisterInitialComponents │  
        // └─┬─┬───────────────────────────────────────────────────┘  
        //   │ │                                                      
        //   │ └──IncrementalRegisterComponents(NumComponentsToUpdate)
        //   │                                                        
        //   │                                                        
        // ┌─▼────────────────────────────────────┐                   
        // │ EIncrementalComponentState::Finalize │                   
        // └──────────────────────────────────────┘   
        //
        switch (IncrementalComponentState)
        {
        case EIncrementalComponentState::Init:
            // sort actors to ensure that parent actors will be registered before child actors
            // haker: before registering components for all actors in Level, sort actors hierarchically
            SortActorsHierarchy(Actors, this);
            IncrementalComponentState = EIncrementalComponentState::RegisterInitialComponents;
            // haker: NOTE that no break expression here!
         
        // PreRegister 단계에서는 별다른 처리는 없고
        // 초기화가 잘 이루어져있늦지 정도만 파악
				case EIncrementalComponentState::PreRegisterInitialComponents:
						if (IncrementalPreRegisterComponents(Context))
						{
							IncrementalComponentState = EIncrementalComponentState::RegisterInitialComponents;
							bAllowLoop = true;
						}
						break;
						
        case EIncrementalComponentState::RegisterInitialComponents:
            if (IncrementalRegisterComponents(true, NumComponentsToUpdate, Context))
            {
#if WITH_EDITOR || 1
								// 에디터가 아니라면 이미 BP가 쿠킹된 이후이므로 RunConstructionScripts 단계가 실행될 필요가 없음
                // haker: SCS is editor-specific running logic
                const bool bShouldRunConstructionScripts = !bHasRerunConstructionScripts && bRerunConstructionScripts && !IsTemplate();
                IncrementalComponentState = bShouldRunConstructionScripts ? EIncrementalComponentState::RunConstructionScripts : EIncrementalComponentState::Finalize; 
#else
                IncrementalComponentState = EIncrementalComponentState::Finalize;
#endif`
            }
            break;
#if WITH_EDITOR || 1
        // haker: we are not going to look into SCS related codes
        case EIncrementalComponentState::RunConstructionScripts:
            if (IncrementalRunConstructionScripts(bFullyUpdateComponents))
            {
                IncrementalComponentState = EIncrementalComponentState::Finalize;
            }
            break;
#endif
        case EIncrementalComponentState::Finalize:
            // haker: we finish registering components for all actors in Level
            IncrementalComponentState = EIncrementalComponentState::Init;
            CurrentActorIndexForIncrementalUpdate = 0;
            bHasCurrentActorCalledPreRegister = false;
            bAreComponentsCurrentlyRegistered = true;
            // haker: it is GC related feature, so I skip this for now
            CreateCluster();
            break;
        }
    
    // haker: focus the condition:
    // - we only iterate again only if bFullyUpdateComponents is 'true'
    } while (bFullyUpdateComponents && !bAreComponentsCurrentlyRegistered);
    
    // haker: process all pending physics state creation
    // - by processing deferred creation of physics state, all components are reflected to Physics World (FPhysScene)
    {
        FPhysScene* PhysScene = OwningWorld->GetPhysicsScene();
        if (PhysScene)
        {
            PhysScene->ProcessDeferredCreatePhysicsState();
        }
    }
}
```

#### Foundation - CreateWorld - SortActorsHierarchy(Level.cpp)

```cpp
static void SortActorsHierarchy(TArray<TObjectPtr<AActor>>& Actors, ULevel* Level)
{
    // haker: we covered conceptually how child-actor works in AActor-UActorComponent structure
    // - this function sorts actor's depth considering AActor-AActor parent-child relationships
    TMap<AActor*, int32> DepthMap;
    // TInlineAllocator => 정해진 사이즈 만큼은 힙으로 할당하는 것이 아닌 내부적으로(스택)에 공간을 할당하고 그 이상 벗어나면 힙에 할당하는 할당자  
    TArray<AActor*, TInlineAllocator<10>> VistedActors;

    DepthMap.Reserve(Actors.Num());

    bool bFoundWorldSettings = false;

                                                                                                                                        
    TFunction<int32(AActor*)> CalcAttachDepth = [
        &DepthMap, 
        &VisitedActors, 
        // haker: we capture TFunction CalcAttachDepth to call lambda recursively
        &CalcAttachDepth, 
        &bFoundWorldSettings]
        (AActor* Actor)
    {
        int32 Depth = 0;
        // haker: not need to do it again if we found the depth of Actor
        if (int32* FoundDepth = DepthMap.Find(Actor))
        {
            Depth = *FoundDepth;
        }
        else
        {
		        // WorldSettings는 Persistent Level에 생성 됨
            // WorldSettings is expected to be the first element in the sorted Actors array
            // to accomodate for the known issue where two world settings can exist, we only sort the
            // first one we find to the 0 index
            // haker: as we saw previously, AWorldSettings is the first Actor added when we create UWorld
            // - we need to make sure AWorldSetting is in index-0
            if (Actor->IsA<AWorldSettings>())
            {
                if (!bFoundWorldSettings)
                {
                    // haker: by setting AWorldSetting's depth as lowest value in int32, we can guarantee that AWorldSetting is in index-0
                    Depth = TNumericLimits<int32>::Lowest();
                    bFoundWorldSettings = true;
                }
                else
                {
                    UE_LOG(LogLevel, Warning, TEXT("Detected duplicate WorldSettings actor - UE-62934"));
                }
            }
            else if (AActor* ParentActor = Actor->GetAttachParentActor())
            {
                if (!VisitedActors.Contains(ParentActor))
                {
                    VisitedActors.Add(Actor);

                    // Actors attached to a ChildActor have to be registered first or else
                    // they will become detached due to the AttachChildren not yet being populated
                    // and thus not recorded in the ComponentInstanceDataCache
                    if (ParentActor->IsChildActor())
                    {
                        // haker: this case is kind of exception handling... BUT, still can't come up with proper scenario...
                        // *** I need to experiment how it works ?!
                        Depth = CalcAttachDepth(ParentActor) - 1;
                    }
                    else
                    {
                        // haker: 
                        //       ┌──────────────────────────────────────────────────────────┐                                                                           
                        //       │                                                          │                                                                           
                        //       │      Actor0 ◄───  Depth = 0                              │                                                                           
                        //       │       │                                                  │                                                                           
                        //       │       └─RootComponent                                    │                                                                           
                        //       │          │                                               │                                                                           
                        //       │          └─Component1                                    │                                                                           
                        //       │             │                                            │                                                                           
                        //       │             └─Actor1 ◄───  Depth = 1                     │                                                                           
                        //       │                │                                         │                                                                           
                        //       │                └─RootComponent                           │                                                                           
                        //       │                   │                                      │                                                                           
                        //       │                   └─Actor2 ◄────  Depth = 2              │                                                                           
                        //       │                                                          │                                                                           
                        //       │                                                          │                                                                           
                        //       └──────────────────────────────────────────────────────────┘                                                                           
                        Depth = CalcAttachDepth(ParentActor) + 1;
                    }
                }
            }
            DepthMap.Add(Actor, Depth);
        }
        return Depth;
    };
    // haker: iterating Actors in ULevel, calculate depth with respect to AActor (not ActorComponent!)
    for (AActor* Actor : Actors)
    {
        if (Actor)
        {
            CalcAttachDepth(Actor);
            VisitedActors.Reset();
        }
    }

    auto DepthSorter = [&DepthMap](AActor* A, AActor* B)
    {
        const int32 DepthA = A ? DepthMap.FindRef(A) : MAX_int32;
        const int32 DepthB = B ? DepthMap.FindRef(B) : MAX_int32;
        return DepthA < DepthB;
    };

    // haker: merge sort:
    // - leave it to read the code for you ~:)
    // 최종적으로 병합 정렬을 하기 위해 그룹을 나눠서 버블 정렬을 하고
    // 그 결과물들을 병합 정렬하는 방식으로 작동함
    StableSortInternal(Actors.GetData(), Actors.Num(), DepthSorter);

    // since all the null entries got sorted to the end, loop them off right now
    // haker: after sorting, there are remaining entries in Actors array, so remove them
    int32 RemoveAtIndex = Actors.Num();
    while (RemoveAtIndex > 0 && Actors[RemoveAtIndex - 1] == nullptr)
    {
        --RemoveAtIndex;
    }
    if (RemoveAtIndex < Actors.Num())
    {
        Actors.RemoveAt(RemoveAtIndex, Actors.Num() - RemoveAtIndex);
    }
}
```

다음과 같이 Actor의 뎁스를 가지고 Scene Graph를 생성

```cpp
// 
// haker: we need to understand how child actor's root-component is attached its parent-actor:
// - in the last time, I just simply mention how child actor works in unreal
// - the actual way is related to SceneComponent's AttachParent and AttachChildren:
//                                                                                                                                              
//       ┌──────────┐                                                               ┌──────────┐                                                
//       │  Actor0  ├──────────────────────────────────────────────────────────────►│  Actor1  │                                                
//       └──┬───────┘                                                               └─┬────────┘                                                
//          │                                                                         │                                                         
//       ┌─RootComponent─────────────────────────────────┐                         ┌─RootComponent─────────────────────────────────┐            
//       │ AttachParent: NULL                            │                         │ AttachParent: Actor0's Component1             │            
//       │ Children: [Component1, Component3]            │               ┌─────────┤ Children: [Component1, Component2]            │            
//       └──┬────────────────────────────────────────────┘               │         └──┬────────────────────────────────────────────┘            
//          │                                                       Actor0──Actor1    │                                                         
//          │ ┌─Component1────────────────────────────────────┐          │            │ ┌─Component1────────────────────────────────────┐       
//          ├─┤ AttachParent: RootComponent                   │◄─────────┘            ├─┤ AttachParent: RootComponent                   │       
//          │ │ Children: [Actor1's RootComponent,Component2] │                       │ │ Children: []                                  │       
//          │ └─┬─────────────────────────────────────────────┘                       │ └───────────────────────────────────────────────┘       
//          │   │                                                                     │                                                         
//          │   │  ┌─Component2────────────────────────────────────┐                  │ ┌─Component2────────────────────────────────────┐       
//          │   └──┤ AttachParent: Component1                      │                  └─┤ AttachParent: RootComponent                   │       
//          │      │ Children: []                                  │                    │ Children: []                                  │       
//          │      └───────────────────────────────────────────────┘                    └───────────────────────────────────────────────┘       
//          │                                                                                                                                   
//          │   ┌─Component3────────────────────────────────────┐                                                                               
//          └───┤ AttachParent: RootComponent                   │                                                                               
//              │ Children: []                                  │                                                                               
//              └───────────────────────────────────────────────┘                                                                               
//      
```

#### Foundation - CreateWorld - AActor::GetAttachParentActor(Actor.cpp)

```cpp
AActor* AActor::GetAttachParentActor() const
{
	if (GetRootComponent() && GetRootComponent()->GetAttachParent())
	{
		return GetRootComponent()->GetAttachParent()->GetOwner();
	}

	return nullptr;
}
```

#### Foundation - CreateWorld - AActor::IsChildActor(Actor.cpp)

```cpp
bool AActor::IsChildActor() const
{
	return ParentComponent.IsValid();
}

// AActor's member variable
UPROPERTY()
TWeakObjectPtr<UChildActorComponent> ParentComponent;
```

#### Foundation - CreateWorld - ULevel::IncrementalRegisterComponents(Level.cpp)

```cpp
bool IncrementalRegisterComponents(bool bPreRegisterComponents, int32 NumComponentsToUpdate, FRegisterComponentContext* Context)
{
    // find next valid actor to process components registration
    // haker: CurrentActorIndexForIncrementalUpdate is persistent index to keep track of the last index which we have done in IncrementalRegisterComponents
    while (CurrentActorIndexForIncrementalUpdate < Actors.Num())
    {
        AActor* Actor = Actors[CurrentActorIndexForIncrementalUpdate];
        bool bAllComponentsRegistered = true;
        if (IsValid(Actor))
        {
            // haker: bPreRegisterComponent is whether we call PreRegisterComponents for each Actor in Level
            if (bPreRegisterComponents && !bHasCurrentActorCalledPreRegister)
            {
                // haker: remember AActor's PreRegisterAllComponent() call in here
                Actor->PreRegisterAllComponents();
                bHasCurrentActorCalledPreRegister = true;
            }

            // haker: here, we register component in the incremental manner
            bAllComponentsRegistered = Actor->IncrementalRegisterComponents(NumComponentsToUpdate, Context);
        }

        // haker: when we successfully register all components in AActor, we prepare next actor
        if (bAllComponentsRegistered)
        {
            // all components have been registered for this actor, move to a next one
            CurrentActorIndexForIncrementalUpdate++;
            bHasCurrentActorCalledPreRegister = false;
        }

        // if we do an incremental registration return to outer loop after each processed actor
        // so outer loop can decide whether we want to continue processing this frame
        // haker: what does this condition means?
        // - we do NOT modify NumComponentsToUpdate in AActor::IncrementalRegisterComponents()
        // - NumComponentsToUpdate != 0 is that we are going to do incremental-update
        //   - when incremental-update is enabled, we are get out of while-loop every actor
        // - ULevel keep track of current actor index to do incremental-update with CurrentActorIndexForIncrementalUpdate
        // - the comment describes:
        //   - ***outer loop calling this function*** determines whether we are going to continue do incremental-update for this Level
        //   - see the src to understand : 
        //     Level->IncrementalUpdateComponents in World.cpp
        //       - it determines by how much time do we left to do incremental-update
        if (NumComponentsToUpdate != 0)
        {
            break;
        }
    }

    // haker: we successfully done to do all incremental-updates for this Level, and return 'true'
    if (CurrentActorIndexForIncrementalUpdate >= Actors.Num())
    {
        // we need to process pending adds prior to rerunning the construction scripts which may internally preform removals / adds themselves
        if (Context)
        {
            Context->Process();
        }
        CurrentActorIndexForIncrementalUpdate = 0;
        return true;
    }

    return false;
}
```

#### Foundation - CreateWorld - AActor::IncrementalRegisterComponents(Actor.cpp)

```cpp
/** incrementally registers components associated with this actor, used during level streaming */
bool IncrementalRegisterComponents(int32 NumComponentsToRegister, FRegisterComponentContext* Context = nullptr)
{
    if (NumComponentsToRegister == 0)
    {
        // 0 - means register all components
        NumComponentsToRegister = MAX_int32;
    }

    UWorld* const World = GetWorld();

    // if we are not a game world, then register tick functions now, if we are a game world we wait until right before BeginPlay()
    // so as to not actually tick until BeginPlay() executes (which could otherwise happen in network games)
    // haker: PIE is GameWorld!
    if (!World->IsGameWorld())
    {
        // haker: in our case, we analyze the engine code in editor environment, let's get into this function
        // - note that bDoComponents is false!
        // - when debugging in PIE, we will not enter this function:
        //   - BUT, lets look into it
        RegisterAllActorTickFunctions(true, false);
    }

		// 무조건 RootComponent부터 설정 시작
    // register RootComponent first so all other children components can reliably use it (i.e., call GetLocation) when they register
    if (RootComponent != nullptr && !RootComponent->IsRegistered())
    {
        if (RootComponent->bAutoRegister)
        {
            // Component 등록을 진행하기 전에 Transactional 퍼버에 담아서 undo처리를 할 수 있도록 준비?
            // before we register our component, save it to our transaction buffer so if "undone" it will return to an unregistered state
            // this should prevent unwanted components hanging around when undoing a copy/paste or duplication action
            // haker: we skip to get into Modify() function:
            // - Modify() will add history (or information) into transaction buffer
            // - transaction buffer is used to deal with undo operation
            RootComponent->Modify(false);

						// LineBatcher 뿐만 아니라 ActorComponent의 등록은 World 단위?
						// 하긴 랜더링 월드, 피직스 월드에 등록해야하는 단위가 월드인가
            RootComponent->RegisterComponentWithWorld(World, Context);
        }
    }
    
    // see GetComponents (goto 077)
    // see template GetComponents (goto 078)
    TInlineComponentArray<UActorComponent*> Components;
	  GetComponents(Components);

    // haker:
    // NumTotalRegisteredComponents - includes previous registered components
    // NumRegisteredComponentsThisRun - do not include previous registered components:
    //   - incremental-update variable to limit by NumComponentsToRegister
    int32 NumTotalRegisteredComponents = 0;
    int32 NumRegisteredComponentsThisRun = 0;

    // haker: we start from 0 to Components.Num()
    TSet<UActorComponent*> RegisteredParents;
    for (int32 CompIdx = 0; CompIdx < Components.Num() && NumRegisteredComponentsThisRun < NumComponentsToRegister; ++CompIdx)
    {
        UActorComponent* Component = Components[CompIdx];

        // haker: we skip already registered ActorComponent:
        // - we do incremental-update, so it could be already registered
        if (!Component->IsRegistered() && Component->bAutoRegister && IsValidChecked(Component))
        {
            // ensure that all parents are registered first
            // see GetUnregisteredParent (goto 081)
            USceneComponent* UnregisteredParentComponent = GetUnregisteredParent(Component);

            // haker: if all parent components are registered, UnregisteredParentComponent will be nullptr
            if (UnregisteredParentComponent)
            {
                bool bParentAlreadyHandled = false;
                // haker: see RegisteredParents, now we can understand what this variable is for
                RegisteredParents.Add(UnregisteredParentComponent, &bParentAlreadyHandled);
                if (bParentAlreadyHandled)
                {
                    UE_LOG(LogActor, Error, TEXT("AActor::IncrementalRegisterComponents parent component '%s' cannot be registered in actor '%s'"), *GetPathNameSafe(UnregisteredParentComponent), *GetPathName());
                    break;
                }
                
                Component = UnregisteredParentComponent;
                CompIdx--;
                NumTotalRegisteredComponents--; // because we will try to register the parent again later
            }

            // before we register our component, save it to our transaction buffer so if 'undone' it will return to an unregistered state
            // This should prevent unwanted components hanging around when undoing a copy/paste or duplication action.
            Component->Modify(false);

            Component->RegisterComponentWithWorld(World, Context);
            NumRegisteredComponentsThisRun++;
        }

        NumTotalRegisteredComponents++;
    }

    // see whether we are done
    if (Components.Num() == NumTotalRegisteredComponents)
    {
        // haker: we are DONE with incremental-component-update!
        bHasRegisteredAllComponents = true;
        // finally, call PostRegisterAllComponents
        PostRegisterAllComponents();
        // after all components have been registered the actor is considered fully added; notify the owning world
        World->NotifyPostRegisterAllActorComponents(this);
        return true;
    }

    return false;
}
```

```cpp
// register parent first, then return to this component on a next iteration
// haker: let's think about how we process unregistered parent
// - can you imagine how this works?
// - Diagrams:                                                                                                                              
//   ┌───AActor::OwnedComponents─────────────────────────────────────────────────────────────────────────────────────────────────────┐
//   │                                                                                                                               │
//   │                                                                                                                               │
//   │                                                    ┌────AttachParent───┐                                                      │
//   │                                                    │                   │                                                      │
//   │                                                    │                   │                                                      │
//   │       ┌────────────┐     ┌────────────┐      ┌─────┴──────┐      ┌─────▼──────┐      ┌────────────┐       ┌────────────┐      │
//   │       │ Component0 ├─────┤ Component1 ├──────┤ Component2 ├──────┤ Component3 ├──────┤ Component4 ├───────┤ Component5 │      │
//   │       └─────┬──────┘     └────────────┘      └─────▲──────┘      └─────┬──────┘      └────────────┘       └──────▲─────┘      │
//   │             │                                      │                   │                                         │            │
//   │             │                                      │                   │                                         │            │
//   │             └──────────────AttachParent────────────┘                   └──────────────AttachParent───────────────┘            │
//   │                                                                                                                               │
//   │                                                                                                                               │
//   │                                                                                                                               │
//   └───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
//
//    1. CompIdx == 0
//       - Component == Component0 (Components[0])
//       - UnregisteredParentComponent == Component2
//       - CompIdx == -1
//       - NumTotalRegisteredComponents == -1
//       - NumRegisteredComponentsThisRun == 1
//    2. CompIdx == 0 (0 = -1 + 1)
//       - Component == Component0 (Components[0])
//       - UnregisteredParentComponent == Component3
//       - CompIdx == -1
//       - NumTotalRegisteredComponents == -1
//       - NumRegisteredComponentsThisRun == 2
//    3. CompIdx == 0 (0 = -1 + 1)
//       - Component == Component0 (Components[0])
//       - UnregisteredParentComponent == Component5
//       - CompIdx == -1
//       - NumTotalRegisteredComponents == -1
//       - NumRegisteredComponentsThisRun == 3
//    4. CompIdx == 0 (0 = -1 + 1)
//       - Component == Component0 (Components[0])
//       - UnregisteredParentComponent == nullptr **** finally can update Component0
//       - CompIdx == 0
//       - NumTotalRegisteredComponents == 1
//       - NumRegisteredComponentsThisRun == 4
//    5. CompIdx == 1 
//       - Component == Component1 (Components[1])
//       - UnregisteredParentComponent == nullptr
//       - CompIdx == 1
//       - NumTotalRegisteredComponents == 2
//       - NumRegisteredComponentsThisRun == 5
//    6. CompIdx == 2
//       - Component == Component2 (Components[2]) **** it is already updated
//       - NumTotalRegisteredComponents == 3
//       - NumRegisteredComponentsThisRun == 5 *** we are NOT update component, so count is remained
//    7. CompIdx == 3
//       - Component == Component3 (Components[3]) **** it is already updated
//       - NumTotalRegisteredComponents == 4
//       - NumRegisteredComponentsThisRun == 5 *** we are NOT update component, so count is remained

```

#### Foundation - CreateWorld - AActor::RegisterAllActorTickFunctions(Actor.cpp)

```cpp
/** when called, will call the virtual call chain to register all of the tick functions for both the actor and optionally all components */
void RegisterAllActorTickFunctions(bool bRegister, bool bDoComponents)
{
		// CDO 혹은 ArcheType
    if (!IsTemplate())
    {
        // prevent repeated redundant attempts
        if (bTickFunctionsRegistered != bRegister)
        {
            FActorThreadContext& ThreadContext = FActorThreadContext::Get();

            RegisterActorTickFunctions(bRegister);
            bTickFunctionsRegistered = bRegister;

            // haker: validate TestRegisterTickFunctions updated in RegisterActorTickCuntions, then reset it
            // 왜 굳이 이 단계를 TLS 변수를 써가면서 체크하지? 어차피 게임 쓰레드에서만 접근가능하지 않나?
            check(ThreadContext.TestRegisterTickFunctions == this);
            ThreadContext.TestRegisterTickFunctions = nullptr;
        }

        // haker: remember we set bDoComponents as false in our previous callstack
        if (bDoComponents)
        {
            for (UActorComponent* Component : GetComponents())
            {
                if (Component)
                {
                    Component->RegisterAllComponentTickFunctions(bRegister);
                }
            }
        }

        if (bAsyncPhysicsTickEnabled)
        {
            //...
        }

        // search (goto 074)
    }
}
```

#### Foundation - CreateWorld - class FActorThreadContext(Actor.cpp)

```cpp
/** thread safe container for actor related global variables */
// haker: 
// - what is TLS(Thread-Local Storage)?
// - why is it thread-safe?
// - understand TLS with TThreadSingleton
// - VERY USEFUL concept to use parallel programming in client-side
// TThreadSingleton : TLS인 싱글톤
class FActorThreadContext : public TThreadSingleton<FActorThreadContext>
{
		friend TThreadSingleton<FActorThreadContext>;

    FActorThreadContext()
        : TestRegisterTickFunctions(nullptr)
    {}

    /** tests tick function registration */
    AActor* TestRegisterTickFunctions;
};
```

#### Foundation - CreateWorld - AActor::RegisterActorTickFunctions(Actor.cpp)

```cpp
/** virtual call chain to register all tick functions for the actor class hierarchy */
virtual void RegisterActorTickFunctions(bool bRegister)
{
    if (bRegister)
    {
        if (PrimaryActorTick.bCanEverTick)
        {
            PrimaryActorTick.Target = this;
            PrimaryActorTick.SetTickFunctionEnable(PrimaryActorTick.bStartsWithTickEnabled || PrimaryActorTick.IsTickFunctionEnabled());
            PrimaryActorTick.RegisterTickFunction(GetLevel());
        }
    }
    else
    {
        if (PrimaryActorTick.IsTickFunctionRegistered())
        {
            PrimaryActorTick.UnRegisterTickFunction();
        }
    }

    // haker: cache the current Actor's TickFunction in TLS
    // - this TLS variable could be used in successive virtual call 
    // 영문을 모를 코드 사용...
    FActorThreadContext::Get().TestRegisterTickFunctions = this;
}
```
