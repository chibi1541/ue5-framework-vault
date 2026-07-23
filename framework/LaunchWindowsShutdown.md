---
related:
  - "[[WinMain]]"
tags:
  - LaunchWindows_cpp
---

```cpp
// 1 - Foundation - Entry - LaunchWindowsShutdown
void LaunchWindowsShutdown()
{
    FEngineLoop::AppExit();
}
```

## 설명
- `FEngineLoop::AppExit()`을 호출하여 애플리케이션 종료 처리를 수행한다.
