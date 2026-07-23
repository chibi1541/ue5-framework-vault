---
related:
  - "[[UEngine/UEngine|UEngine]]"
  - "[[UEditorEngine/UEditorEngine.GetEditorWorldContext|UEditorEngine::GetEditorWorldContext]]"
  - "[[UEditorEngine/UEditorEngine.Tick|UEditorEngine::Tick]]"
tags:
  - EditorEngine_h
---

```cpp
class UEditorEngine : public UEngine
{
public:
    /** returns the WorldContext for the editor world. for now, there will always be exactly 1 of these in the editor */
    FWorldContext& GetEditorWorldContext(bool bEnsureIsGWorld = false)
    {
        for (int32 i = 0; i < WorldList.Num(); ++i)
        {
            if (WorldList[i].WorldType == EWorldType::Editor)
            {
                return WorldList[i];
            }
        }

        // haker: if we get into this line of code, it will crashed:
        // - where is initial editor world created?
        // - the hint is provided by the below comment
        check(false); // there should have already been one created in ***UEngine::Init***

        return CreateNewWorldContext(EWorldType::Editor);
    }
};
```

## 설명
- `UEngine`을 상속한 에디터 전용 엔진 클래스.
- `WorldList`는 부모인 `UEngine`이 들고 있으며, 그중 `EWorldType::Editor` 컨텍스트를 찾아 반환한다.
