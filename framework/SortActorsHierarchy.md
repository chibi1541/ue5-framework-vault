---
related:
  - "[[ULevel/ULevel|ULevel]]"
  - "[[AActor/AActor|AActor]]"
  - "[[ULevel/ULevel.IncrementalUpdateComponents|ULevel::IncrementalUpdateComponents]]"
  - "[[AActor/AActor.GetAttachParentActor|AActor::GetAttachParentActor]]"
  - "[[AActor/AActor.IsChildActor|AActor::IsChildActor]]"
  - "[[USceneComponent/USceneComponent|USceneComponent]]"
tags:
  - Level_cpp
---

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

```cpp
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

## 설명
- [[ULevel/ULevel.IncrementalUpdateComponents|IncrementalUpdateComponents()]]의 `Init` 단계에서 호출된다. 부모 액터가 자식 액터보다 먼저 register 되도록 `Actors` 배열을 depth 기준으로 정렬한다.
- 정렬 기준이 되는 depth는 `UActorComponent`가 아니라 **AActor-AActor 부모-자식 관계**로 계산한다. 액터끼리의 부모 관계는 `UChildActorComponent`와 [[USceneComponent/USceneComponent|USceneComponent]]의 `AttachParent`/`AttachChildren`를 통해 성립하며, [[AActor/AActor.GetAttachParentActor|GetAttachParentActor()]]로 역추적한다.
- `CalcAttachDepth`는 `TFunction` 자기 자신을 캡처해 재귀 호출한다(람다를 재귀로 쓰기 위한 관용구). `DepthMap`에 이미 계산된 액터는 건너뛴다.
- `AWorldSettings`는 `UWorld` 생성 시 가장 먼저 추가되는 액터이므로 정렬 후에도 index-0이어야 한다. depth를 `TNumericLimits<int32>::Lowest()`로 주어 이를 보장한다. WorldSettings가 중복 존재할 수 있는 알려진 이슈(UE-62934) 때문에, 처음 발견한 하나만 0번으로 보낸다.
- 부모가 [[AActor/AActor.IsChildActor|IsChildActor()]]인 경우에는 depth를 `+1`이 아니라 `-1` 해 더 먼저 등록되게 한다. ChildActor에 붙은 액터들은 `AttachChildren`이 아직 채워지지 않아 detach 되어버릴 수 있기 때문이다.
- `TInlineAllocator<10>`은 지정한 크기까지는 스택(컨테이너 내부)에 할당하고, 그것을 넘어서면 힙으로 넘어가는 allocator다.
- `StableSortInternal`은 그룹을 나눠 버블 정렬한 뒤 그 결과들을 병합 정렬하는 방식으로 동작한다. 정렬 후 뒤쪽에 몰린 `nullptr` 엔트리는 잘라낸다(`MAX_int32` depth로 취급되므로 끝으로 밀린다).
