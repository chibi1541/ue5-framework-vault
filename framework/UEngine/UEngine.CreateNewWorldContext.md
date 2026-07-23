---
related:
  - "[[UEngine/UEngine.Init|UEngine::Init]]"
  - "[[FWorldContext/FWorldContext|FWorldContext]]"
  - "[[UEngine/UEngine|UEngine]]"
tags:
  - UnrealEngine_cpp
---

```cpp
FWorldContext& CreateNewWorldContext(EWorldType::Type WorldType)
{
    FWorldContext* NewWorldContext = new FWorldContext;
    WorldList.Add(NewWorldContext);
    NewWorldContext->WorldType = WorldType;
    NewWorldContext->ContextHandle = FName(*FString::Printf(TEXT("Context_%d"), NextWorldContextHandle++));
    return *NewWorldContext;
}
```

## 설명
- 새 `FWorldContext`를 생성해 `WorldList`에 추가하고 참조를 반환한다.
- `ContextHandle`은 `NextWorldContextHandle`을 증가시키며 `Context_%d` 형태로 부여된다.
