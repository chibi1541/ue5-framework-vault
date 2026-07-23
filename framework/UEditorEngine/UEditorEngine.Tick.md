---
related:
  - "[[FEngineLoop/FEngineLoop.Tick|FEngineLoop::Tick]]"
  - "[[UEditorEngine/UEditorEngine.GetEditorWorldContext|UEditorEngine::GetEditorWorldContext]]"
  - "[[UEditorEngine/UEditorEngine|UEditorEngine]]"
tags:
  - EditorEngine_cpp
---

```cpp
virtual void Tick(float DeltaSeconds, bool bIdleMode) override
{
    // haker: where does GWorld is updated initially?
    UWorld* CurrentGWorld = GWorld;

    FWorldContext& EditorContext = GetEditorWorldContext();
}
```

## 설명
- `UEngine`을 상속받은 `UEditorEngine`의 Tick. (에디터를 중점으로 분석하기 때문에 이쪽으로 이동)
- `GetEditorWorldContext()`로 에디터 월드의 `FWorldContext`를 가져온다.
