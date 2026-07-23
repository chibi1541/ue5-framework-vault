---
related:
  - "[[EngineTick]]"
  - "[[UEditorEngine/UEditorEngine.Tick|UEditorEngine::Tick]]"
tags:
  - LaunchEngineLoop_cpp
---

```cpp
virtual void Tick() override
{
    // set FApp::CurrentTime, FApp::DeltaTime and potentially wait to enforce max tick rate
    {
		    // 엔진에는 다양한 쓰레드들(피직스, RHI 등등)이 있는데
		    // 이 쓰레드들 간의 싱크를 맞추는 작업을 아래의 함수에서 실행
		    // 그렇기 때문에 매우 스케일이 크다고 함;;
    
        // haker: as the comments say, it provides CurrentTime, DeltaTime:
        // - however, the most important thing is that this function syncs the GameThread and RenderTherad including RHIThread
        // - these threads are running in parallel:
        //   - to prevent one of these threads running pass over GameThread, the variables like FrameNumber are used to sync each other
        GEngine->UpdateTimeAndHandleMaxTickRate();
    }
    
    // beginning of RHI frame
		{
			ENQUEUE_RENDER_COMMAND(BeginFrame)([CurrentFrameCounter](FRHICommandListImmediate& RHICmdList)
			{
				BeginFrameRenderThread(RHICmdList, CurrentFrameCounter);
			});

			for (FSceneInterface* Scene : GetRendererModule().GetAllocatedScenes())
			{
				ENQUEUE_RENDER_COMMAND(FScene_StartFrame)([Scene](FRHICommandListImmediate& RHICmdList)
				{
					Scene->StartFrame();
				});
			}

			UE::RenderCommandPipe::StartRecording();

#if !UE_SERVER && WITH_ENGINE
			if (!GIsEditor && GEngine->GameViewport && GEngine->GameViewport->GetWorld() && GEngine->GameViewport->GetWorld()->IsCameraMoveable())
			{
				// When not in editor, we emit dynamic resolution's begin frame right after RHI's.
				GEngine->EmitDynamicResolutionEvent(EDynamicResolutionStateEvent::BeginFrame);
			}
#endif
		}
    
    // Calculates average FPS/MS (outside STATS on purpose)
		CalculateFPSTimings();

		// OS의 메시지 큐 처리(키보드/마우스 입력 처리는 이때 처리됨)
		FPlatformApplicationMisc::PumpMessages(true);

		// process accumulated Slate input
		if (FSlateApplication::IsInitialized() && !bIdleMode)
		{
			FSlateApplication& SlateApp = FSlateApplication::Get();
			
			// 게임 패드 같은 디바이스 입력 메시지를 처리
      SlateApp.PollGameDeviceState();
      
			// Gives widgets a chance to process any accumulated input
      SlateApp.FinishedInputThisFrame();
		}
		
    // main game engine tick (world, game objects, etc)
    // haker:
    // - in editor (lyra project), call the following order: ULyraEditorEngine -> UUnrealEdEngine -> UEditorEngine
    // - we get into UEditorEngine directly
    GEngine->Tick(FApp::GetDeltaTime(), bIdleMode);
}
```

## 설명
- `UpdateTimeAndHandleMaxTickRate()` — `FApp::CurrentTime` / `FApp::DeltaTime`을 갱신하는 동시에, 병렬로 도는 GameThread·RenderThread·RHIThread 간 동기화를 담당한다. 한 쓰레드가 GameThread를 앞질러 가지 않도록 `FrameNumber` 같은 변수로 sync를 맞추기 때문에 스케일이 매우 크다.
- RHI 프레임 시작: `ENQUEUE_RENDER_COMMAND`로 렌더 스레드에 `BeginFrame`, `FScene_StartFrame`을 큐잉한다.
- `FPlatformApplicationMisc::PumpMessages(true)` — OS 메시지 큐 처리. 키보드/마우스 입력이 이때 처리된다.
- Slate 입력 처리: `PollGameDeviceState()`로 게임 패드 등 디바이스 입력을, `FinishedInputThisFrame()`으로 위젯에 누적 입력 처리 기회를 준다.
- `GEngine->Tick()` — world와 game object의 본 tick. 에디터(Lyra 기준)에서는 `ULyraEditorEngine` → `UUnrealEdEngine` → `UEditorEngine` 순으로 호출된다.
