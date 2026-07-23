---
related:
  - "[[GuardedMain]]"
  - "[[FEngineLoop/FEngineLoop.Tick|FEngineLoop::Tick]]"
tags:
  - Launch_cpp
---

```cpp
void EngineTick()
{
    // see FEngineLoop::Tick
    // 여기서부터 본격적으로 Tick 시작
    GEngineLoop.Tick();
}
```

## 설명
- `GuardedMain`의 메인 루프에서 매 프레임 호출된다.
- 실제 Tick 로직은 `FEngineLoop::Tick`에서 시작된다.
