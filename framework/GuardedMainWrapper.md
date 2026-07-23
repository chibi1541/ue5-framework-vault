---
related:
  - "[[LaunchWindowsStartup]]"
  - "[[GuardedMain]]"
tags:
  - LaunchWindows_cpp
---

```cpp
int32 GuardedMainWrapper(const TCHAR* CmdLine)
{
    int32 ErrorLevel = 0;
    // see GuardMain()
    ErrorLevel = GuardedMain(CmdLine);
    return ErrorLevel;
}
```

## 설명
- SEH `__try` 블록 안에서 호출되는 wrapper. 실제 로직은 `GuardedMain`에 있다.
