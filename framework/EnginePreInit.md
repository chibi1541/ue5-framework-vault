---
related:
  - "[[GuardedMain]]"
  - "[[FEngineLoop/FEngineLoop.PreInit|FEngineLoop::PreInit]]"
tags:
  - Launch_cpp
---

```cpp
// 싱글톤 객체로 엔진의 루프는 얘가 관장함
FEngineLoop GEngineLoop;

int32 EnginePreInit(const TCHAR* CmdLine)
{
    int32 ErrorLevel = GEngineLoop.PreInit(CmdLine);
    return ErrorLevel;
}
```

## 설명
- `GEngineLoop`은 싱글톤 객체로 엔진의 루프를 관장한다.
- 실제 구현은 `FEngineLoop::PreInit`.
