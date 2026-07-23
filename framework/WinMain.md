---
related:
  - "[[LaunchWindowsStartup]]"
  - "[[LaunchWindowsShutdown]]"
tags:
  - LaunchWindows_cpp
---

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

## 설명
- Windows 플랫폼에서 엔진의 진입점(entry point).
- startup / shutdown 두 단계로만 구성되어 있다.
