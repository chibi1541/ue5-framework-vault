---
related:
  - "[[UEditorEngine/UEditorEngine|UEditorEngine]]"
  - "[[FWorldContext/FWorldContext|FWorldContext]]"
  - "[[UEngine/UEngine.Init|UEngine::Init]]"
  - "[[UEngine/UEngine.CreateNewWorldContext|UEngine::CreateNewWorldContext]]"
tags:
  - Engine_h
---

```cpp
// 앤진도 UObject를 상속한 객체
// 즉, GC에 의해 라이프 사이클이 제어되는 객체임
class UEngine : public UObject, public FExec
{
		// 이게 월드 -> 월드를 WorldContext라는 단위로 관리함, 그 주체가 Engine
    // haker: recommend to look into TIndirectArray, focusing on differences from TArray
    // see FWorldContext
    // TIndirectArray => typedef TArray<void*, Allocator>
    // 저장 객체를 포인터로 관리, 소유권에 대한 이야기도 하는데
    // 배열이 사라질 때 안에 들어가 있는 포인터를 정리해주기 때문에 나온 이야기 같음
    // 배열이지만 포인터를 담기 때문에 배열의 연속성으로 인한 캐시 최적화는 기대할수 없다고...
    // 그렇지만 복사 비용이 큰 객체를 담기 위함과 이들의 리소스 관리를 위해 활용된다고 함
    TIndirectArray<FWorldContext> WorldList;
    int32 NextWorldContextHandle;
};
```

## 설명
- 엔진도 `UObject`를 상속한 객체다. 즉 GC에 의해 라이프 사이클이 제어된다.
- 엔진은 월드를 `FWorldContext` 단위로 관리하며, 그 목록이 `WorldList`다.
- `TIndirectArray`는 저장 객체를 포인터로 관리하고 배열이 사라질 때 내부 포인터를 정리해준다. 포인터를 담기 때문에 배열 연속성으로 인한 캐시 최적화는 기대할 수 없지만, 복사 비용이 큰 객체를 담고 리소스를 관리하는 데 활용된다.
