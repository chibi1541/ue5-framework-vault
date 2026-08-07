---
related:
  - "[[FEngineLoop/FEngineLoop.Tick|FEngineLoop::Tick]]"
  - "[[UEditorEngine/UEditorEngine.GetEditorWorldContext|UEditorEngine::GetEditorWorldContext]]"
  - "[[UEditorEngine/UEditorEngine|UEditorEngine]]"
  - "[[UWorld/UWorld.Tick|UWorld::Tick]]"
  - "[[Enum/ELevelTick|ELevelTick]]"
tags:
  - EditorEngine_cpp
---

```cpp
virtual void Tick(float DeltaSeconds, bool bIdleMode) override
{
    // haker: where does GWorld is updated initially?
    UWorld* CurrentGWorld = GWorld;

    FWorldContext& EditorContext = GetEditorWorldContext();
}
```

```cpp
virtual void Tick(float DeltaSeconds, bool bIdleMode) override
{
    // haker: where does GWorld is updated initially?
    UWorld* CurrentGWorld = GWorld;

    FWorldContext& EditorContext = GetEditorWorldContext();
    
    // viewport 관련 초기화 처리
    
    // by default we tick the editor world
    bool bShouldTickEditorWorld = true;

    ELevelTick TickType = IsRealTime ? LEVELTICK_ViewportOnly : LEVELTICK_TimeOnly;
    // PIE가 아닌 에디터 월드에서는 파티클 정도만 Tick이 돔
    // 실제 Tick이 돌려면 PIE 환경이 되어야 함
    if (bShouldTickEditorWorld)
    {
        // NOTE: still allowing the FX system to tick so particle systems don't restart after entering/leaving responsive mode
        // haker: FX(niagara) system should be running even in level-viewport mode
        {
            // haker: here is EditorWorld for level-viewport ticking:
            // - level-viewport's world is ticking in very limited allowance
            // - so, we look into world's tick with PIE Context
            // 지금 이건 Editor World임 밑에 PIE World가 있음
            EditorContext.World()->Tick(TickType, DeltaSeconds);
        }
    }

    {
        // determine number of PIE worlds that should tick 
        TArray<FWorldContext*> LocalPieContextPtrs;
        for (FWorldContext& PieContext : WorldList)
        {
            // haker: EWorldType::PIE!
            if (PieContext.WorldType == EWorldType::PIE && PieContext.World() != nullptr)
            {
                LocalPieContextPtrs.Add(&PieContext);
            }
        }

		    // CreateInnerProcessPIEGameInstance()는 나중에 확인 
        // haker: when is WorldContext for PIE initialized? 
        // - when you want to know exactly what happen on this: see the callstack after setting BP(breakpoint) to CreateInnerProcessPIEGameInstance
        // see UEditorEngine::CreateInnerProcessPIEGameInstance()
        // - for now, we are looking into how tick function runs first, then we go back to see how PIE is initialized

        for (FWorldContext* PieContextPtr : LocalPieContextPtrs)
        {
            // haker: after we see CreateInnerProcessPIEGameInstance(), then we understand what GameViewport is for
            FWorldContext& PieContext = *PieContextPtr;
            PlayWorld = PieContext.World();
            GameViewport = PieContext.GameViewport;

            /** how much time to use per tick */
            // haker: nothing special, it allows to run in fixed fps with repsect to DeltaSeconds
            float TickDeltaSeconds;
            if (PieContext.PIEFixedTickSeconds > 0.f)
            {
                PieContext.PIEAccumulatedTickSeconds += DeltaSeconds;
                TickDeltaSeconds = PieContext.PIEFixedTickSeconds;
            }
            else
            {
                // haker: otherwise, we will get into here (one tick per frame)
                PieContext.PIEAccumulatedTickSeconds = DeltaSeconds;
                TickDeltaSeconds = DeltaSeconds;
            }

            for (; PieContext.PIEAccumulatedTickSeconds >= TickDeltaSeconds; PieContext.PIEAccumulatedTickSeconds -= TickDeltaSeconds)
            {
                // update the level
                {
                    // tick the level
                    PieContext.World()->Tick(LEVELTICK_All, TickDeltaSeconds);
                }
            }
        }
    }
}
```

## 설명
- `UEngine`을 상속받은 `UEditorEngine`의 Tick. (에디터를 중점으로 분석하기 때문에 이쪽으로 이동)
- `GetEditorWorldContext()`로 에디터 월드의 `FWorldContext`를 가져온다.
- 에디터 월드는 기본적으로 매 프레임 tick하지만([[Enum/ELevelTick|ELevelTick]]), realtime 뷰포트면 `LEVELTICK_ViewportsOnly`, 아니면 `LEVELTICK_TimeOnly`로만 돈다. 즉 파티클/FX 정도만 갱신되고 액터 tick은 돌지 않는다. 실제 게임 로직 tick을 보려면 PIE 환경이어야 한다.
- responsive mode를 드나들 때 파티클이 재시작되지 않도록, level-viewport 모드에서도 FX(Niagara) system은 계속 tick된다.
- 그 다음 `WorldList`에서 `EWorldType::PIE`이면서 월드가 살아있는 컨텍스트만 모아 PIE 월드들을 tick한다. 이때 `PlayWorld`/`GameViewport`를 해당 PIE 컨텍스트의 것으로 교체한다.
- `PIEFixedTickSeconds`가 설정되어 있으면 누적 시간(`PIEAccumulatedTickSeconds`)을 고정 간격으로 나눠 여러 번 tick하고, 아니면 프레임당 한 번 tick한다.
- PIE 월드는 `LEVELTICK_All`로 [[UWorld/UWorld.Tick|UWorld::Tick]]을 호출하므로 모든 게임 루프가 진행된다.
- PIE용 `FWorldContext`가 언제 초기화되는지는 `UEditorEngine::CreateInnerProcessPIEGameInstance()`에 breakpoint를 걸어 콜스택으로 확인할 수 있다. (나중에 확인)
