---
related:
  - "[[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]"
  - "[[UActorComponent/UActorComponent.ExecuteRegisterEvents|UActorComponent::ExecuteRegisterEvents]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[AActor/AActor|AActor]]"
tags:
  - ActorComponent_cpp
---

```cpp
/** registers a component with a specific world, which creates any visual/physical state */
void RegisterComponentWithWorld(UWorld* InWorld, FRegisterComponentContext* Context = nullptr)
{
    // if the component was already registered, do nothing
    if (IsRegistered())
    {
        return;
    }

    // haker: it is natural to early-out cuz there is no world to register
    // 이 구문이 있다는 건 어딘가에 World가 생성되기 전에 Register가 호출 될 수도 있는 건가?
    if (InWorld == nullptr)
    {
        return;
    }

    // if not registered, should not have a scene
    // haker: it means that we can know whether it is registered or not by its existance of WorldPrivate
    check(WorldPrivate == nullptr);

    // haker: UWorld::LineBatcher(ULineBatchComponent)'s owner is nullptr
    AActor* MyOwner = GetOwner();
    checkSlow(MyOwner == nullptr || MyOwner->OwnsComponent(this));

    if (!HasBeenCreated)
    {
        // haker: do you remember OnComponentCreated() event covered in AActor?
        // - it is called when ActorComponent is being created
        OnComponentCreated();
    }

    WorldPrivate = InWorld;

    // 여기서 각 월드(랜더링, 피직스)의 분신(State)를 생성
    ExecuteRegisterEvents(Context);

    // if not in a game world register ticks now, otherwise defer until BeginPlay
    // if no owner we won't trigger BeginPlay() either so register now in that case as well
    if (!InWorld->IsGameWorld())
    {
        // haker: 
        // - if the world is not game world, we didn't call InitializeComponent()
        //   - see the condition below (MyOwner == nullptr) 
        RegisterAllComponentTickFunctions(true);
    }
    else if (MyOwner == nullptr)
    {
        // haker: here is the InitializeComponent() call which we covered in AActor's initialization stages
        if (!bHasBeenInitialized && bWantsInitializeComponent)
        {
            InitializeComponent();
        }

        RegisterAllComponentTickFunctions(true);

        // haker: here is the exact case matching UWorld::LineBatcher
    }
    else
    {
        // haker: this is **the NORMAL case** we expect
        MyOwner->HandleRegisterComponentWithWorld(this);
    }

    // if this is a blueprint created component and it has component children they can miss getting registered in some scenarios
    if (IsCreatedByConstructionScript())
    {
        // haker: explain what is SCS(Simple Construction Script) and CCS(Custom Construction Script) with examples (or with editor)
        // - Children will be collected from SCS and CCS, and they have outer object as this UActorComponent
        TArray<UObject*> Children;
        GetObjectsWithOuter(this, Children, true, RF_NoFlags, EInternalObjectFlags::Garbage);

        // haker: iterating Children, and try to RegisterComponentWithWorld (recursive calls)
        for (UObject* Child : Children)
        {
            if (UActorComponent* ChildComponent = Cast<UActorComponent>(Child))
            {
                if (ChildComponent->bAutoRegister && !ChildComponent->IsRegistered() && ChildComponent->GetOwner() == MyOwner)
                {
                    ChildComponent->RegisterComponentWithWorld(InWorld);
                }
            }
        }
    }
}

// 컴포넌트 생성 시에 무언가 처리를 넣고 싶으면 이걸 오버라이딩
/** called when a component is created (not loaded); this can happen in the editor or during gameplay */
virtual void OnComponentCreated()
{
    ensure(!bHasBeenCreated);
    bHasBeenCreated = true;
}
```

## 설명
- LineBatcher처럼 World에 바로 종속된 `UActorComponent`를 register 하기 위한 함수. 특정 월드에 컴포넌트를 등록하면서 visual/physical state를 생성한다.
- 등록 여부는 `WorldPrivate`의 존재로 판별한다. 등록되지 않았다면 `WorldPrivate == nullptr`이어야 한다.
- `bHasBeenCreated`가 false면 `OnComponentCreated()`를 호출한다. 컴포넌트 생성 시점에 처리를 넣고 싶으면 이 가상 함수를 오버라이딩한다.
- `WorldPrivate` 대입 후 [[UActorComponent/UActorComponent.ExecuteRegisterEvents|UActorComponent::ExecuteRegisterEvents]]를 호출해 render world/physics world의 분신(state)을 생성한다.
- tick function 등록 분기가 세 가지다:
  - game world가 아니면 즉시 `RegisterAllComponentTickFunctions(true)` (이 경우 `InitializeComponent()`는 호출되지 않는다).
  - game world이고 owner가 nullptr이면 `InitializeComponent()` 후 tick function을 등록한다. `UWorld::LineBatcher`가 정확히 이 경우다.
  - owner가 있으면 `MyOwner->HandleRegisterComponentWithWorld(this)`로 넘긴다. 이것이 일반적으로 기대하는 경로다.
- construction script(SCS/CCS)로 생성된 컴포넌트는 outer가 자신인 자식 컴포넌트들이 등록에서 빠질 수 있으므로, `GetObjectsWithOuter`로 자식을 모아 재귀적으로 `RegisterComponentWithWorld`를 호출한다.
