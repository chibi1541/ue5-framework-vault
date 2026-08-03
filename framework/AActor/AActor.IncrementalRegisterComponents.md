---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[USceneComponent/USceneComponent|USceneComponent]]"
  - "[[ULevel/ULevel.IncrementalRegisterComponents|ULevel::IncrementalRegisterComponents]]"
  - "[[AActor/AActor.RegisterAllActorTickFunctions|AActor::RegisterAllActorTickFunctions]]"
  - "[[UActorComponent/UActorComponent.RegisterComponentWithWorld|UActorComponent::RegisterComponentWithWorld]]"
tags:
  - Actor_cpp
---

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
            // Component 등록을 진행하기 전에 Transactional 버퍼에 담아서 undo처리를 할 수 있도록 준비?
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

## 설명
- 액터 하나에 속한 `UActorComponent`들을 등록하는 함수. `NumComponentsToRegister`에 0이 들어오면 `MAX_int32`로 바꿔 전부 등록한다.
- game world가 **아닐 때만** [[AActor/AActor.RegisterAllActorTickFunctions|RegisterAllActorTickFunctions(true, false)]]를 여기서 호출한다. game world(PIE 포함)에서는 `BeginPlay()` 직전까지 미뤄서, `BeginPlay()` 전에 tick이 도는 것을 막는다. 이때 `bDoComponents`가 false라는 점에 주의.
- 등록은 반드시 `RootComponent`부터 시작한다. 다른 자식 컴포넌트들이 등록되면서 `GetLocation()` 같은 호출로 RootComponent에 의존할 수 있기 때문이다.
- `Modify(false)`는 등록 전에 컴포넌트를 transaction buffer에 기록해, undo 시 unregistered 상태로 되돌아가게 한다. 복사/붙여넣기나 복제를 undo했을 때 컴포넌트가 남아도는 것을 막는 장치다.
- 실제 등록은 [[UActorComponent/UActorComponent.RegisterComponentWithWorld|RegisterComponentWithWorld()]]로 이루어진다. 컴포넌트의 등록 단위가 world인 이유는, render world(`FScene`)와 physics world(`FPhysScene`)에 state를 추가해야 하기 때문이다.
- 카운터가 두 개인 이유: `NumTotalRegisteredComponents`는 이전에 이미 등록된 것까지 포함한 누적치(완료 판정용)이고, `NumRegisteredComponentsThisRun`은 이번 호출에서 새로 등록한 개수(증분 제한용)다.
- `GetUnregisteredParent()`로 아직 등록되지 않은 부모 [[USceneComponent/USceneComponent|USceneComponent]]가 있으면, 그 부모를 먼저 등록하고 `CompIdx--`로 같은 인덱스를 다시 돌린다. 부모 체인을 타고 올라가며 등록한 뒤 원래 컴포넌트로 돌아오는 구조다. `RegisteredParents`는 같은 부모가 두 번 걸리는(= 순환/이상 상태) 경우를 감지해 에러 로그를 남기고 빠져나오기 위한 집합이다.
- `Components.Num() == NumTotalRegisteredComponents`이면 모두 등록된 것이므로 `bHasRegisteredAllComponents`를 세우고 `PostRegisterAllComponents()`와 `World->NotifyPostRegisterAllActorComponents()`를 호출한 뒤 `true`를 반환한다.
