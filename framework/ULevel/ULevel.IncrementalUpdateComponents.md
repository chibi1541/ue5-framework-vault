---
related:
  - "[[ULevel/ULevel|ULevel]]"
  - "[[ULevel/ULevel.UpdateLevelComponents|ULevel::UpdateLevelComponents]]"
  - "[[SortActorsHierarchy|SortActorsHierarchy]]"
  - "[[ULevel/ULevel.IncrementalRegisterComponents|ULevel::IncrementalRegisterComponents]]"
tags:
  - Level_cpp
---

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
        // 초기화가 잘 이루어져있는지 정도만 파악
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
#endif
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

## 설명
- `NumComponentsToUpdate == 0`이면 `bFullyUpdateComponents`가 true가 되어, 레벨의 전 액터/액터 컴포넌트가 한 번에 register된다. 전체 과정 중 실제로 증분적으로 진행되는 것은 `RegisterInitialComponents`와 `RunConstructionScripts` 단계뿐이다.
- 증분 업데이트(`NumComponentsToUpdate != 0`)는 game world에서만 허용된다. 에디터 월드는 항상 0을 넘겨야 한다는 것을 두 개의 `check()`로 강제한다.
- `IncrementalComponentState`([[ULevel/ULevel|ULevel]]의 `EIncrementalComponentState`)를 state machine처럼 돌린다: `Init` → `PreRegisterInitialComponents` → `RegisterInitialComponents` → (에디터 한정) `RunConstructionScripts` → `Finalize`.
  - `Init`: [[SortActorsHierarchy|SortActorsHierarchy()]]로 부모 액터가 자식 액터보다 먼저 등록되도록 정렬한다. `break`가 없어 그대로 다음 case로 fall-through 된다.
  - `PreRegisterInitialComponents`: 별다른 처리 없이 초기화가 제대로 되었는지 확인하는 수준.
  - `RegisterInitialComponents`: [[ULevel/ULevel.IncrementalRegisterComponents|IncrementalRegisterComponents()]]로 실제 등록을 수행한다.
  - `RunConstructionScripts`: SCS는 editor 전용 로직이다. 쿠킹된 빌드에서는 BP가 이미 처리되어 있으므로 이 단계가 필요 없다.
  - `Finalize`: state를 `Init`으로 되돌리고 인덱스/플래그를 리셋한 뒤 `bAreComponentsCurrentlyRegistered = true`로 마무리한다.
- `do-while` 조건이 `bFullyUpdateComponents && !bAreComponentsCurrentlyRegistered`이므로, 한 번에 전부 갱신하는 경우에만 `Finalize`에 도달할 때까지 루프를 반복한다.
- 마지막으로 `FPhysScene::ProcessDeferredCreatePhysicsState()`를 호출해, 지연되어 있던 physics state 생성을 한 번에 처리한다. 이로써 등록된 컴포넌트들이 physics world에 반영된다.
