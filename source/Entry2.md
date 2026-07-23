---
summarize: true
---

<aside>  
📝

## Tick

#### // 4 - Foundation - Entry - EngineTick(Launch.cpp)

```cpp
void EngineTick()
{
    // see FEngineLoop::Tick
    // 여기서부터 본격적으로 Tick 시작
    GEngineLoop.Tick();
}
```

#### // 9 - Foundation - Entry - FEngineLoop::Tick

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

#### // 10 - Foundation - Entry - UEditorEngine::Tick(EditorEngine.cpp)

FEngine을 상속받은 UEditorEngine의 Tick으로 이동(에디터를 중점으로 볼거기 때문에)

```cpp
virtual void Tick(float DeltaSeconds, bool bIdleMode) override
{
    // haker: where does GWorld is updated initially?
    UWorld* CurrentGWorld = GWorld;

    FWorldContext& EditorContext = GetEditorWorldContext();
}
```

- 잠깐 UEngine 클래스를 살펴봄
    
    <aside>  
    💡
    
    #### // 11 - Foundation - Entry - class UEditorEngine(EditorEngine.h)
    
    ```cpp
    class UEditorEngine : public UEngine
    {
    public:
        /** returns the WorldContext for the editor world. for now, there will always be exactly 1 of these in the editor */
        FWorldContext& GetEditorWorldContext(bool bEnsureIsGWorld = false)
        {
            for (int32 i = 0; i < WorldList.Num(); ++i)
            {
                if (WorldList[i].WorldType == EWorldType::Editor)
                {
                    return WorldList[i];
                }
            }
    
            // haker: if we get into this line of code, it will crashed:
            // - where is initial editor world created?
            // - the hint is provided by the below comment
            check(false); // there should have already been one created in ***UEngine::Init***
    
            return CreateNewWorldContext(EWorldType::Editor);
        }
    };
    ```
    
    #### // 12 - Foundation - Entry - class UEngine(Engine.h)
    
    ```cpp
    // 앤진도 UObject를 상속한 객체
    // 즉, GC에 의해 라이프 사이클이 제어되는 객체임
    class UEngine : public UObject, public FExec
    {
    		// 이게 월드 -> 월드를 WorldContext라는 단위로 관리함, 그 주체가 Engine
        // haker: recommend to look into TIndirectArray, focusing on differences from TArray
        // see FWorldContext
        // TIndirectArray => typedef TArray<void*, Allocator>
        // 저장 객체를 포인터로 관리, 소유권에 대한 이야기도 하는데
        // 배열이 사라질 때 안에 들어가 있는 포인터를 정리해주기 때문에 나온 이야기 같음
        // 배열이지만 포인터를 담기 때문에 배열의 연속성으로 인한 캐시 최적화는 기대할수 없다고...
        // 그렇지만 복사 비용이 큰 객체를 담기 위함과 이들의 리소스 관리를 위해 활용된다고 함
        TIndirectArray<FWorldContext> WorldList;
        int32 NextWorldContextHandle;
    };
    ```
    
    #### // 13 - Foundation - Entry - struct FWorldContext
    
    <aside>  
    ✏️
    
    WorldContext 주석
    
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
    
    </aside>
    
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
    
    #### // 14 - Foundation - Entry - enum EWorldType(EngineTypes.h)
    
    ```cpp
    // haker: note that it is useful pattern define enum in UE5
    // - if you wrap it up with namespace, you can avoid enum name conflicts with other global enum
    namespace EWorldType
    {
        enum Type
        {
    				/** An untyped world, in most cases this will be the vestigial worlds of streamed in sub-levels */
    				None,
    		
    				/** The game world */
    				Game,
    		
    				/** A world being edited in the editor */
    				Editor,
    		
    				/** A Play In Editor world */
    				PIE,
    		
    				/** A preview world for an editor tool */
    				EditorPreview,
    		
    				/** A preview world for a game */
    				GamePreview,
    		
    				/** A minimal RPC world for a game */
    				GameRPC,
    		
    				/** An editor world that was loaded but not currently being edited in the level editor */
    				Inactive
        };
    }
    ```
    
    </aside>
    

#### // 16 - Foundation - Entry - UEditorEngine::GetEditorWorldContext(EditorEngine.cpp)

```cpp
/** returns the WorldContext for the editor world. for now, there will always be exactly 1 of these in the editor */
FWorldContext& GetEditorWorldContext(bool bEnsureIsGWorld = false)
{
    for (int32 i = 0; i < WorldList.Num(); ++i)
    {
		    // 스태틱 메쉬를 보여주는 월드는 EditorPreview 타입이기 때문에
		    // 보통 에디터 월드는 EWorldType::Editor 하나 밖에 없음
        if (WorldList[i].WorldType == EWorldType::Editor)
        {
            return WorldList[i];
        }
    }

    // haker: if we get into this line of code, it will crashed:
    // - where is initial editor world created?
    // - the hint is provided by the below comment
    check(false); // there should have already been one created in ***UEngine::Init***
    // see UEngine::Init()

    return CreateNewWorldContext(EWorldType::Editor);
}
```

#### // 17 - Foundation - Entry - UEngine::Init(UnrealEngine.cpp)

지금까지 당연하게 GWorld 을 호출했는데 그럼 GWorld 은 어디서 만들어지냐? 에 대한 답

```cpp
virtual void Init(IEngineLoop* InEngineLoop)
{
    if (GIsEditor)
    {
        // create a WorldContext for the editor to use and create an initially empty world
        // haker: here, we make sure that at least, one editor world exists
        FWorldConext& InitialWorldContext = CreateNewWorldContext(EWorldType::Editor);

				// 여기서 최초의 EditorWorld를 생성
        // haker: we get into Foundation - CreateWorld
        InitialWorldContext.SetCurrentWorld(UWorld::CreateWorld(EWorldType::Editor, true));
        GWorld = InitialWorldContext.World();
    }
}
```

#### // 17 - Foundation - Entry - UEngine::CreateNewWorldContext(UnrealEngine.cpp)

```cpp
FWorldContext& CreateNewWorldContext(EWorldType::Type WorldType)
{
    FWorldContext* NewWorldContext = new FWorldContext;
    WorldList.Add(NewWorldContext);
    NewWorldContext->WorldType = WorldType;
    NewWorldContext->ContextHandle = FName(*FString::Printf(TEXT("Context_%d"), NextWorldContextHandle++));
    return *NewWorldContext;
}
```

</aside>