---
related:
  - "[[UWorld/UWorld.Tick|UWorld::Tick]]"
  - "[[UEditorEngine/UEditorEngine.Tick|UEditorEngine::Tick]]"
tags:
  - EngineBaseTypes_h
---

```cpp
/** type of tick we wish to perform on the level */
enum ELevelTick
{
    /** update the level time only */
	  // 액터의 Tick(게임 로직)은 돌지 않고 게임 시간만 흐름
    LEVELTICK_TimeOnly = 0,
    /** update time and viewports */
    // 그래픽적인 요소(카메라, 랜더링)만 Tick 이벤트가 진행
    LEVELTICK_ViewportsOnly = 1,
    /** update all */
    // 모든 게임 루프가 진행
    LEVELTICK_All = 2,
    /** delta time is zero, we are paused; components don't tick */
    // 모든 시간이 정지(게임 시간 자체가 흐르지 않음)
    LEVELTICK_PauseTick = 3,
};
```

## 설명
- 레벨에 대해 어떤 수준의 tick을 돌릴지 지정하는 enum. [[UWorld/UWorld.Tick|UWorld::Tick]]의 첫 인자로 넘어간다.
- `LEVELTICK_TimeOnly`: 액터 Tick(게임 로직)은 돌지 않고 게임 시간만 흐른다.
- `LEVELTICK_ViewportsOnly`: 카메라/렌더링 같은 그래픽 요소만 Tick이 진행된다.
- `LEVELTICK_All`: 모든 게임 루프가 진행된다. PIE 월드는 이 값으로 tick한다.
- `LEVELTICK_PauseTick`: delta time이 0인 일시정지 상태로, 컴포넌트가 tick하지 않는다.
- 에디터 월드는 `LEVELTICK_ViewportsOnly`/`LEVELTICK_TimeOnly`로만 돌기 때문에 실제 게임 로직 tick을 보려면 PIE 환경이어야 한다.
