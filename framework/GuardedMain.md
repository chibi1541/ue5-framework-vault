---
related:
  - "[[GuardedMainWrapper]]"
  - "[[LaunchWindowsStartup]]"
  - "[[EnginePreInit]]"
  - "[[EditorInit]]"
  - "[[EngineTick]]"
tags:
  - Launch_cpp
---

```cpp
/**
 * static guarded main function; rolled into own function so we can have error handling for debug/release builds depending on
 * whether a debugger is attached or not
 */
int32 GuardedMain(const TCHAR* CmdLine)
{
    //...

#if !(UE_BUILD_SHIPPING)
    // haker: passing command arguments with "-waitforattach","-WaitForDebugger"  is useful when you want to debug at the starting of the engine
		// If "-waitforattach" or "-WaitForDebugger" was specified, halt startup and wait for a debugger to attach before continuing

		const bool bWaitForDebuggerAndBreak = FParse::Param(CmdLine, TEXT("waitforattach")) || FParse::Param(CmdLine, TEXT("WaitForDebugger")); 
		const bool bWaitForDebugger = bWaitForDebuggerAndBreak || FParse::Param(CmdLine, TEXT("WaitForAttachNoBreak")) || FParse::Param(CmdLine, TEXT("WaitForDebuggerNoBreak"));

		if (bWaitForDebugger)
		{
			while (!FPlatformMisc::IsDebuggerPresent())
			{
				// haker: when debugger is attached, it will be out of inf-loop
				FPlatformProcess::Sleep(0.1f);
			}

			if (bWaitForDebuggerAndBreak)
			{
				// haker: it stops here:
        // - it is VERY useful to debugging the VERY starting point
				UE_DEBUG_BREAK();
			}
		}
#endif

    // super early init code: DO NOT MOVE THIS ANYWHERE ELSE:
    // haker:
    // - CoreDelegates, CoreUObjectDelegates, WorldDelegates, ... 
    // - these delegate classes are good to remember
    // - the unreal engine gives a way to inject the code in a form of provding delegate class like above
    // - here, you **can inject the code very starting point of the unreal engine**
    FCoreDelegates::GetPreMainInitDelegate().Broadcast();

    // make sure GEngineLoop::Exit() is always called:
    // haker: RAII -> this pattern is frequently used in unreal engine
    struct EngineLoopCleanupGuard
    {
        ~EngineLoopCleanupGuard()
        {
            EngineExit();
        }
    } CleanupGuard;

    // see EnginePreInit()
    int32 ErrorLevel = EnginePreInit(CmdLine);

    {
    // 여기서 에디터냐 엔진이냐로 나뉘는데 해당 강의에서는 에디터에 초점
    // 디버깅이 용이하기 때문, 물론 두 로직이 그렇게까지 차이나지 않는다고...
    // 오히려 에디터가 더 복잡하게 초기화 할게 많음
#if WITH_EDITOR || 1
        if (GIsEditor)
        {
            // haker:
            // - what we are focusing on is the editor build to analyze engine code
            // see EditorInit()
            ErrorLevel = EditorInit(GEngineLoop);
        }
        else
#endif
        {
            ErrorLevel = EngineInit();
        }
    }

    // haker: the pattern to calculate the elapsed time:
    // int64 elapedTime = ::GetTickCount64() - startTime; <- 요런거임
    // double GStartTime = FPlatformTime::Secons();
    double EngineInitializationTime = FPlatformTime::Seconds() - GStartTime;

    // haker: IsEngineExitRequested() == GIsRequestingExit
    // - here is main loop of the engine
    while (!IsEngineExitRequested())
    {
        // haker: before diving into huge code of EngineTick, let's finished reading rest of code in GuardMain() briefly
        // see EngineTick()
        EngineTick();
    }

#if WITH_EDITOR || 1
    if (GIsEditor)
    {
        EditorExit();
    }
#endif

    return ErrorLevel;
}
```

## 설명
- 해당 함수가 엔진 메인 루프.
- `-waitforattach`, `-WaitForDebugger` command line 인자는 엔진 시작 시점부터 디버깅할 때 유용하다.
- `FCoreDelegates::GetPreMainInitDelegate()` — 언리얼 엔진의 가장 이른 시점에 코드를 주입할 수 있는 delegate. `CoreDelegates`, `CoreUObjectDelegates`, `WorldDelegates` 같은 delegate class는 기억해 둘 것.
- `EngineLoopCleanupGuard`는 RAII 패턴으로 `EngineExit()` 호출을 보장한다. 언리얼 엔진에서 자주 쓰이는 패턴.
- Editor 빌드냐 게임 빌드냐에 따라 `EditorInit` / `EngineInit`으로 갈린다. (에디터가 초기화할 것이 더 많고 복잡하지만 디버깅이 용이함)
- `IsEngineExitRequested()`가 `true`가 될 때까지 `EngineTick()`을 반복하는 것이 메인 루프.
