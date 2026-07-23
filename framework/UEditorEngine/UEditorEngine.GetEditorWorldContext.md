---
related:
  - "[[UEditorEngine/UEditorEngine.Tick|UEditorEngine::Tick]]"
  - "[[UEngine/UEngine.Init|UEngine::Init]]"
  - "[[FWorldContext/FWorldContext|FWorldContext]]"
  - "[[UEditorEngine/UEditorEngine|UEditorEngine]]"
tags:
  - EditorEngine_cpp
---

```cpp
/** returns the WorldContext for the editor world. for now, there will always be exactly 1 of these in the editor */
FWorldContext& GetEditorWorldContext(bool bEnsureIsGWorld = false)
{
    for (int32 i = 0; i < WorldList.Num(); ++i)
    {
		    // 스태틱 메쉬를 보여주는 월드는 EditorPreview 타입이기 때문에
		    // 보통 에디터 월드는 EWorldType::Editor 하나 밖에 없음
        if (WorldList[i].WorldType == EWorldType::Editor)
        {
            return WorldList[i];
        }
    }

    // haker: if we get into this line of code, it will crashed:
    // - where is initial editor world created?
    // - the hint is provided by the below comment
    check(false); // there should have already been one created in ***UEngine::Init***
    // see UEngine::Init()

    return CreateNewWorldContext(EWorldType::Editor);
}
```

## 설명
- `WorldList`를 순회하며 `EWorldType::Editor` 타입의 context를 찾아 반환한다.
- 스태틱 메쉬를 보여주는 월드는 `EditorPreview` 타입이라, 보통 에디터 월드는 `Editor` 하나뿐이다.
- 못 찾으면 `check(false)`로 크래쉬한다. 최초 에디터 월드는 `UEngine::Init`에서 이미 만들어져 있어야 하기 때문.
