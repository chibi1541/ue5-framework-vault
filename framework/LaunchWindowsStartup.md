---
related:
  - "[[WinMain]]"
  - "[[GuardedMainWrapper]]"
  - "[[GuardedMain]]"
tags:
  - LaunchWindows_cpp
---

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

## 설명
- `GuardedMain`인 이유는 크래쉬 발생 시에 예외처리(SEH)를 위해 메인을 감싸는 함수이기 때문이다.
- command line 처리: 언리얼 엔진은 여기서 command line을 자신들의 형식으로 변환한다.
- debugger가 attach 되어 있거나 `-noexceptionhandler` 옵션이 있으면 SEH 없이 곧바로 `GuardedMain` 실행.
- 그 외에는 SEH(structured exception handling)로 감싸서 크래쉬를 trap하고 crash report를 생성한다.
