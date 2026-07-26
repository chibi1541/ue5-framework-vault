---
summarize: true
---

## 시작(LaunchWindows.cpp, Launch.cpp)

#### // 0 - Foundation - Entry - BEGIN(LaunchWindows.cpp)

```cpp
// 엔진의 시작점 
/** haker: this is the entry in windows platform in the engine */
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)
{
	int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr);
	LaunchWindowsShutdown();
	return Result;
}
```

#### // 2 - Foundation - Entry - LaunchWindowsStartup(LaunchWindows.cpp)

GuardedMain인 이유는 크래쉬 발생 시에 예외처리(SEH)를 위해 메인을 감싸는 함수 이므로

```cpp
int32 LaunchWindowsStartup(HINSTANCE hInInstance, HINSTANCE hPrevInstance, char*, int32 nCmdShow, const TCHAR* CmdLine)
{
    int32 ErrorLevel = 0;

    if (!CmdLine)
    {
        // haker://
        // - if you have any experience to write your own small game engine or app, it is natural to proces command line:
        // - unreal engine also process command line here to transform command line in their own flavor
        CmdLine = GetCommandLineW();
        if (ProcessCommandLine())
        {
            CmdLine = *GSavedCommandLine;
        }
    }

		if ( FParse::Param( CmdLine,TEXT("crashreports") ) )
		{
				GAlwaysReportCrash = true;
		}
		
		bool bNoExceptionHandler = FParse::Param(CmdLine,TEXT("noexceptionhandler"));
		(void)bNoExceptionHandler;
	
		bool bIgnoreDebugger = FParse::Param(CmdLine, TEXT("IgnoreDebugger"));
		(void)bIgnoreDebugger;
	
		bool bIsDebuggerPresent = FPlatformMisc::IsDebuggerPresent() && !bIgnoreDebugger;
		(void)bIsDebuggerPresent;

		if (GUELibraryOverrideSettings.bIsEmbedded || bNoExceptionHandler || (bIsDebuggerPresent && !GAlwaysReportCrash))
		{
				// Don't use exception handling when a debugger is attached to exactly trap the crash. This does NOT check
				// whether we are the first instance or not!
				// ExceptionHandler를 설정하지 않는다면 SEH 패턴 없이 곧 바로 메인 실행
				ErrorLevel = GuardedMain( CmdLine );
		}
		else
    {
        // use SEH (structured exception handling) to trap the crash:
        // haker: if you don't know what SEH is, plz searching it and make sure to understand this!
        // - one thing I want to point out is that SEH is triggered by OS (kernel mode):
        //   - the way of behaving SEH is not static, it could be changed anytime by OS provider like MS:
        //      - cuz, it's related OS security itself
        //      - if you are interested in this, start from:
        //          - IDTR register
        //          - IDT(interrupt descriptor table)
        //          - ISRs(interrupt service routines)
        //          - interrupt dispatching
        //          - kernel patch protection
        //      - for my recommendation, just skip this, just good to know ~ :)
        __try
        {
            GIsGuarded = 
            // see GuardedMainWrapper()
            ErrorLevel = GuardedMainWrapper(CmdLine);
            GIsGuarded = 0;
        }
        // haker: when exception triggered, catch here
        // - when a crash occurs in the engine, a crash report is generated from here normally
        __except(FPlatformMisc::GetCrashHandlingType == ECrashHandlingType::Default
            ? ReportCrash(GetExceptionInformation())
			: EXCEPTION_CONTINUE_SEARCH)
        {
            ErrorLevel = 1;
            FPlatformMisc::RequestExit(true);
        }
    }

    return ErrorLevel;
}
```

#### // 3 - Foundation - Entry - GuardedMainWrapper(LaunchWindows.cpp)

```cpp
int32 GuardedMainWrapper(const TCHAR* CmdLine)
{
    int32 ErrorLevel = 0;
    // see GuardMain()
    ErrorLevel = GuardedMain(CmdLine);
    return ErrorLevel;
}
```

#### // 4 - Foundation - Entry - GuardedMain(Launch.cpp)

해당 함수가 엔진 메인 루프

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

#### // 1 - Foundation - Entry - LaunchWindowsShutdown(LaunchWindows.cpp)

```cpp
// 1 - Foundation - Entry - LaunchWindowsShutdown
void LaunchWindowsShutdown()
{
    FEngineLoop::AppExit();
}
```

</aside>

<aside>  
📝

## 초기화()

#### // 5 - Foundation - Entry - EnginePreInit(Launch.cpp)

```cpp
// 싱글톤 객체로 엔진의 루프는 얘가 관장함
FEngineLoop GEngineLoop;

int32 EnginePreInit(const TCHAR* CmdLine)
{
    // see FEngineLoop::PreInit()
    int32 ErrorLevel = GEngineLoop.PreInit(CmdLine);
    return ErrorLevel;
}
```

#### // 6 - Foundation - Entry - FEngineLoop::PreInit(LaunchEngineLoop.cpp)

```cpp
/** pre-initialize the main loop - parse command line, sets up GIsEditor, etc */
int32 PreInit(const TCHAR* CmdLine)
{
    //...
}
```

#### // 7 - Foundation - Entry - EditorInit(UnrealEdGlobals.cpp)

```cpp
int32 EditorInit(IEngineLoop& EngineLoop)
{
    int32 ErrorLevel = EngineLoop.Init();
}
```

#### FEngineLoop::Init(LaunchEngineLoop.cpp)

```cpp
int32 FEngineLoop::Init()
{

#if WITH_EDITOR
		GEngine = GEditor = GUnrealEd = NewObject<UUnrealEdEngine>(GetTransientPackage(), EngineClass);
#else

		GEngine->Init(this);
}
```

</aside>