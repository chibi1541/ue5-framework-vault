---
related:
  - "[[UEditorEngine/UEditorEngine.GetEditorWorldContext|UEditorEngine::GetEditorWorldContext]]"
  - "[[UEngine/UEngine.CreateNewWorldContext|UEngine::CreateNewWorldContext]]"
  - "[[UEngine/UEngine|UEngine]]"
tags:
  - UnrealEngine_cpp
---

```cpp
virtual void Init(IEngineLoop* InEngineLoop)
{
    if (GIsEditor)
    {
        // create a WorldContext for the editor to use and create an initially empty world
        // haker: here, we make sure that at least, one editor world exists
        FWorldConext& InitialWorldContext = CreateNewWorldContext(EWorldType::Editor);

				// 여기서 최초의 EditorWorld를 생성
        // haker: we get into Foundation - CreateWorld
        InitialWorldContext.SetCurrentWorld(UWorld::CreateWorld(EWorldType::Editor, true));
        GWorld = InitialWorldContext.World();
    }
}
```

## 설명
- 지금까지 당연하게 호출하던 `GWorld`가 어디서 만들어지는가에 대한 답.
- 에디터일 때 `EWorldType::Editor` context를 만들고, 최초의 EditorWorld를 `UWorld::CreateWorld`로 생성해 `GWorld`에 대입한다.
- 즉 에디터 월드가 최소 하나는 존재함을 여기서 보장한다.
