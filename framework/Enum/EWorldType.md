---
related:
  - "[[FWorldContext/FWorldContext|FWorldContext]]"
tags:
  - EngineTypes_h
---

```cpp
// haker: note that it is useful pattern define enum in UE5
// - if you wrap it up with namespace, you can avoid enum name conflicts with other global enum
namespace EWorldType
{
    enum Type
    {
				/** An untyped world, in most cases this will be the vestigial worlds of streamed in sub-levels */
				None,
		
				/** The game world */
				Game,
		
				/** A world being edited in the editor */
				Editor,
		
				/** A Play In Editor world */
				PIE,
		
				/** A preview world for an editor tool */
				EditorPreview,
		
				/** A preview world for a game */
				GamePreview,
		
				/** A minimal RPC world for a game */
				GameRPC,
		
				/** An editor world that was loaded but not currently being edited in the level editor */
				Inactive
    };
}
```

## 설명
- UE5에서 자주 쓰이는 enum 정의 패턴. namespace로 감싸면 다른 global enum과 이름 충돌을 피할 수 있다.
- 에디터에서 스태틱 메쉬를 보여주는 월드는 `EditorPreview` 타입이라, 보통 에디터 월드는 `Editor` 하나만 존재한다.
