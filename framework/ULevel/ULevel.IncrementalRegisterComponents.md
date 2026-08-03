---
related:
  - "[[ULevel/ULevel|ULevel]]"
  - "[[AActor/AActor|AActor]]"
  - "[[ULevel/ULevel.IncrementalUpdateComponents|ULevel::IncrementalUpdateComponents]]"
  - "[[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents]]"
tags:
  - Level_cpp
---

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

## 설명
- [[ULevel/ULevel.IncrementalUpdateComponents|IncrementalUpdateComponents()]]의 `RegisterInitialComponents` 단계에서 호출된다. 레벨의 `Actors`를 순회하며 액터별로 component 등록을 진행한다.
- `CurrentActorIndexForIncrementalUpdate`는 어디까지 처리했는지를 프레임을 넘어 유지하는 인덱스다. 증분 등록이 중단되었다가 재개될 때 이 인덱스부터 이어서 처리한다.
- `bPreRegisterComponents`가 true면 액터마다 `PreRegisterAllComponents()`를 한 번씩 호출한다. `bHasCurrentActorCalledPreRegister`가 중복 호출을 막고, 액터가 완료되면 다시 false로 리셋된다.
- 실제 등록은 [[AActor/AActor.IncrementalRegisterComponents|AActor::IncrementalRegisterComponents()]]에 위임하고, 그 액터의 컴포넌트가 전부 등록되었을 때만 다음 액터로 넘어간다.
- `NumComponentsToUpdate != 0`이면 액터 하나를 처리할 때마다 while 루프를 빠져나온다. 즉 "이번 프레임에 계속 진행할지"는 이 함수가 아니라 **호출하는 바깥 루프**(World.cpp의 `Level->IncrementalUpdateComponents`, 남은 시간으로 판단)가 결정한다. `NumComponentsToUpdate`는 이 함수 안에서 감소하지 않는다는 점에 주의.
- 모든 액터를 다 처리했다면 `Context->Process()`로 밀려있던 pending add를 먼저 처리한다. ConstructionScript 재실행이 내부적으로 컴포넌트를 추가/제거할 수 있기 때문에 그 전에 정리해두는 것이다. 이후 인덱스를 0으로 되돌리고 `true`를 반환한다.
