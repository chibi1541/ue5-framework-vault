---
related:
  - "[[UEngine/UEngine|UEngine]]"
  - "[[Enum/EWorldType|EWorldType]]"
  - "[[UEditorEngine/UEditorEngine.GetEditorWorldContext|UEditorEngine::GetEditorWorldContext]]"
  - "[[UEngine/UEngine.CreateNewWorldContext|UEngine::CreateNewWorldContext]]"
tags:
  - Engine_h
---

```cpp
/**
 * FWorldContext
 * a context for dealing with UWorlds at the engine level. as the engine brings up and destroys world, we need a way to keep straight
 * what world belongs to what
 * 
 * WorldContexts can be throught of as a track. by default, we have 1 track that we load and unload levels on. adding a second context
 * is adding a second track; another track of progression for worlds to live on.
 * 
 * for the GameEngine, there will be one WorldContext until we decide to support multiple simultaneous worlds.
 * for the EditorEngine, there may be one WorldContext for EditorWorld and one for the PIE World.
 * 
 * FWorldContext provides both a way to manage 'the current PIE UWorld*' as well as state that goes along with connecting/travelling to
 * new worlds
 * 
 * FWorldContext should remain internal to the UEngine classes. outside code should not keep pointers or try to manage FWorldContexts directly.
 * outside code can still deal with UWorld*, and pass UWorld*s into Engine level functions. the Engine code can look up the relevant context
 * for a given UWorld*
 * 
 * for convenience, FWorldContext can maintain outside pointers to UWorld*s. for example, PIE can tie UWorld* UEditorEngine::PlayWorld to the PIE
 * world context. if the PIE UWorld changes, the UEditorEngine::PlayWorld pointer will be automatically updated. this is done with AddRef() and 
 * SetCurrentWorld()
 */
```

```cpp
// haker: 
// - the context for world used in engine level (UEngine, UEditorEngine, etc)
//   - you can think of FWorldContext as descriptor for loosing dependency between engine and world
// - for GameEngine:
//   - usually one WorldContext, when switching worlds like lobby to in-game world, there is the moment when two world contexts exist
// - for EditorEngine:
//   - when we editing the world with level viewport, we normally face one world context
//   - when we execute the PIE, the new world context is generated and simultaneously, two world contexts exists
//   - when you try to run multiplay game in PIE, multiple world contexts can exist
// - as I said, the world context is for engine, so do NOT maintain FWorldContext elsewhere
// 선생님이 이해하신 바로는
// Engine이 World의 생성, 파괴에 직접 관여하지 않는다고 함
// 그렇기 때문에 이 둘 사이의 Dependency를 약하게 할 필요가 있는데
// 이러한 Descriptor(Handle 같은거를 의미함)의 역할을 수행하기 위한게 WorldContext라고 하심
// 이런 WorldContext는 모든 월드에 대한 정보를 가진게 아니고 Engine이 월드를 관리하는데 필요한 정보만 간추려서 가지고 있음
struct FWorldContext
{
    void SetCurrentWorld(UWorld* World)
    {
        UWorld* OldWorld = ThisCurrentWorld;
        ThisCurrentWorld = World;

        if (OwningGameInstance)
        {
            OwningGameInstance->OnWorldChanged(OldWorld, ThisCurrentWorld);
        }
    }

    TEnumAsByte<EWorldType::Type> WorldType;
    
    // haker: assign separate name to context different from UWorld's name
    FName ContextHandle;

    // haker: GameInstance owns ONE world context
    TObjectPtr<class UGameInstance> OwningGameInstance;

    // haker: the world which the context is referencing
    TObjectPtr<UWorld> ThisCurrentWorld;
};
```

## 설명
- 엔진 레벨(`UEngine`, `UEditorEngine` 등)에서 월드를 다루기 위한 context.
- Engine이 World의 생성·파괴에 직접 관여하지 않기 때문에 둘 사이의 dependency를 약하게 할 필요가 있고, 그 descriptor(handle 같은 것) 역할을 하는 것이 WorldContext다. 월드의 모든 정보를 갖는 게 아니라 Engine이 월드를 관리하는 데 필요한 정보만 간추려 갖는다.
- GameEngine: 보통 하나. 로비 → 인게임처럼 월드를 전환하는 순간에는 두 개가 동시에 존재한다.
- EditorEngine: 레벨 뷰포트로 편집할 때는 하나, PIE를 실행하면 새 context가 생겨 두 개가 되고, PIE 멀티플레이에서는 여러 개가 존재할 수 있다.
- `ContextHandle`은 `UWorld`의 이름과는 별개로 context에 부여되는 이름이다.
- 외부 코드는 `FWorldContext`를 직접 들고 있거나 관리하면 안 된다. `UWorld*`만 다루고, 엔진 코드가 해당 월드의 context를 찾아준다.
