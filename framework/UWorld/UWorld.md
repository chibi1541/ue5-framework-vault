---
related:
  - "[[UWorld/UWorld.CreateWorld|UWorld::CreateWorld]]"
  - "[[FWorldInitializationValues/FWorldInitializationValues|FWorldInitializationValues]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[UObject/UObject|UObject]]"
  - "[[Enum/EWorldType|EWorldType]]"
tags:
  - World_h
---

```cpp
/** 
 * the world is the top level object representing a map or a sandbox in which Actors and Components will exist and be rendered 
 * 
 * a world can be a single persistent level with an optional list of streaming levels that are loaded and unloaded via volumes and blueprint functions
 * or it can be a collection of levels organized with a World Composition (->haker: OLD COMMENT...)
 * 
 * in a standalone game, generally only a single World exists except during seamless area transition when both a destination and current world exists
 * in the editor many Worlds exist: 
 * - the level being edited
 * - each PIE instance
 * - each editor tool which has an interactive rendered viewport, and many more
 */
 
class UWorld final : public UObject, public FNetworkNotify
{
		/** the URL that was used when loading this World */
		// haker: think of the URL as package path:
		// - e.g. Game\Map\Seoul\Seoul.umap
		// 파일 Path, 패키지 Path
		FURL URL;
		
		/** the type of world this is. Describes the context in which it is being used (Editor, Game, Preview etc.) */
		// haker: we already seen EWorldType
		// - TEnumAsByte is helper wrapper class to support bit operation on enum type
		// - I recommend to read it how it is implemented
		// ADVICE: as C++ programmer, it is VERY important **to manipulate bit operations freely!**
		// enum 값을 비트 연산하기 위한 랩핑
		TEnumAsByte<EWorldType::Type> WorldType;
		
		/** persistent level containing the world info, default brush and actors pawned during gameplay among other things */
		// hacker: short explanation about world info
		TObjectPtr<class ULevel> PersistentLevel;
}
```

## 설명
- 액터와 컴포넌트가 존재하고 렌더링되는 map/sandbox를 나타내는 최상위 객체. standalone game에서는 보통 하나만 존재하지만, 에디터에서는 편집 중인 레벨, 각 PIE 인스턴스, 뷰포트를 가진 에디터 툴마다 월드가 존재한다.
- `URL`은 패키지 경로(파일 경로)라고 생각하면 된다. (예: `Game\Map\Seoul\Seoul.umap`)
- `WorldType`은 `TEnumAsByte`로 감싸 enum 값에 비트 연산을 쓸 수 있게 한 것.
- `PersistentLevel`은 World Setting에 나오는 World Info를 들고 있는 레벨. 이를 갖지 않는 나머지 레벨들이 SubLevel이다.
