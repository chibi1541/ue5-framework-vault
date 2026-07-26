---
related:
  - "[[UWorld/UWorld|UWorld]]"
  - "[[AActor/AActor|AActor]]"
  - "[[UObject/UObject|UObject]]"
tags:
  - Level_h
---

```cpp
/**
 * the level object:
 * contains the level's actor list, BSP information, and brush list
 * every level has a World as its Outer and can be used as the PersistentLevel, however,
 * when a Level has been streamed in the OwningWorld represents the World that it is a part of 
 */

/**
 * a level is a collection of Actors (lights, volumes, mesh instances etc)
 * multiple levels can be loaded and unloaded into the World to create a streaming experience 
 */

// haker:
// Level?
// - level == collection of actors:
//   - examples of actors:
//     - light, static-mesh, volume, brush(e.g. BSP brush: binary-search-partitioning), ...
//       [ ] explain BSP brush with the editor
// - rest of content will be skipped for now:
//   - when we cover different topics like world-partition, streaming etc, we will visit it again
class ULevel : public UObject
{
		/** array of all actors in this level, used by FActorIteratorBase and derived classes */
		// haker: this is the member variable which contains a list of AActor
		TArray<TObjectPtr<AActor>> Actors;
		
		/** cached level collection that this level is contained in */
		FLevelCollection* CachedLevelCollection;
}
```

## 설명
- 레벨은 액터의 집합(light, static-mesh, volume, BSP brush 등)이다. 액터 리스트, BSP 정보, brush 리스트를 갖는다.
- 모든 레벨은 `UWorld`를 Outer로 가진다. `PersistentLevel`로 쓰일 수도 있고, 스트리밍된 레벨의 경우 `OwningWorld`가 자신이 속한 월드를 가리킨다.
- 여러 레벨을 월드에 load/unload 하면서 streaming 경험을 만든다.
- `Actors`가 이 레벨의 모든 액터를 담고 있으며 `FActorIteratorBase` 계열이 이를 순회한다.
