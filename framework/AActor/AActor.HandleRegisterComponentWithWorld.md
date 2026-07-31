---
related:
  - "[[AActor/AActor|AActor]]"
  - "[[UActorComponent/UActorComponent.RegisterComponentWithWorld|UActorComponent::RegisterComponentWithWorld]]"
  - "[[UActorComponent/UActorComponent.RegisterAllComponentTickFunctions|UActorComponent::RegisterAllComponentTickFunctions]]"
tags:
  - Actor_cpp
---

```cpp
/** finish initializing the component and register tick functions and BeginPlay() if it's the proper time to do so */
void HandleRegisterComponentWithWorld(UActorComponent* Component)
{
    // ActorComponent가 초기화 단게에서만 생성, 등록되는 건 아니기 때문에
    const bool bOwnerBeginPlayStarted = HasActorBegunPlay() || IsActorBeginningPlay();

    // haker: if component is not initialized, try to initialize the component
    if (!Component->HasBeenInitialized() && Component->bWantsInitializeComponent && IsActorInitialized())
    {
        // ActorComponent의 InitializeComponent 함수는 사용자가 오버라이드해서 커스텀하는 것을 주목적으로 설계된 함수
        Component->InitializeComponent();

        // the component was finally initialized, it can now be replicated
        // NOTE that if this component does not ask to be initialized, it would have started to be replicated inside AddOwnedComponent 
        // haker: in editor, if we don't play in PIE, BeginPlay() isn't called
        if (bOwnerBeginPlayStarted)
        {
            // 언리얼 네트워크에서 RPC의 단위가 Actor이기 때문에
            // 여기서 뭔가 처리를 해줌
            // haker: we skip the replication stuff for now
            AddComponentForReplication(Component);
        }
    }

    if (bOwnerBeginPlayStarted)
    {
        // haker: when are we getting into this code?
        // - maybe actor is activated, and it will be triggered, if we add new component to actor which is already activated
        Component->RegisterAllComponentTickFunctions(true);
        if (!Component->HasBegunPlay())
        {
            // 이건 나중에
            Component->BeginPlay();
        }
    }
}
```

## 설명
- [[UActorComponent/UActorComponent.RegisterComponentWithWorld|RegisterComponentWithWorld()]]에서 owner가 있는 **일반적인 경로**로 넘어오는 함수. `UWorld` → `ULevel` → `AActor` → `UActorComponent` 구조를 그대로 따르는 처리 로직이다.
- world에서 곧바로 컴포넌트를 register 하는 경우(예: LineBatcher)는 owner가 없으므로 이 함수가 호출되지 않는다. owner인 `AActor`가 어디까지 초기화되었는지에 따라 컴포넌트의 초기화 정도가 달라지기 때문에, register를 Actor 쪽 함수가 맡는다.
- register 단계 + 필요하다면 `BeginPlay()`까지 한 번에 처리한다.
- 컴포넌트가 아직 초기화되지 않았고 `bWantsInitializeComponent`이며 Actor가 초기화되었다면 `InitializeComponent()`를 호출한다. 이 함수는 사용자가 오버라이드해 커스텀하라고 설계된 함수다.
- `bOwnerBeginPlayStarted`(= `HasActorBegunPlay() || IsActorBeginningPlay()`)인 경우에만 replication 등록, tick function 등록, `BeginPlay()`가 진행된다. 에디터에서 PIE를 돌리지 않으면 `BeginPlay()`는 호출되지 않는다.
- 이미 활성화된 Actor에 새 컴포넌트를 추가하는 상황이 이 경로에 해당한다. 컴포넌트는 초기화 단계에서만 생성/등록되는 것이 아니기 때문이다.
