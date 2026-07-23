---
related:
  - "[[GuardedMain]]"
  - "[[FEngineLoop/FEngineLoop.Init|FEngineLoop::Init]]"
tags:
  - UnrealEdGlobals_cpp
---

```cpp
int32 EditorInit(IEngineLoop& EngineLoop)
{
    int32 ErrorLevel = EngineLoop.Init();
}
```

## 설명
- `GuardedMain`에서 `GIsEditor`가 `true`일 때 호출되는 Editor 빌드 초기화 진입점.
- 내부적으로 `FEngineLoop::Init`을 호출한다.
