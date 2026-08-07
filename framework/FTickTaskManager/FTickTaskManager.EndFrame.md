---
related:
  - "[[UWorld/UWorld.Tick|UWorld::Tick]]"
  - "[[FTickTaskLevel/FTickTaskLevel.EndFrame|FTickTaskLevel::EndFrame]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
tags:
  - TickTaskManager_cpp
---

```cpp
/** Finish a frame of ticks **/
virtual void EndFrame() override
{
	TickTaskSequencer.EndFrame();
	bTickNewlySpawned = false;
	for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
	{
		LevelList[LevelIndex]->EndFrame();
	}

	FTaskSyncManager* SyncManager = FTaskSyncManager::Get();
	if (SyncManager)
	{
		SyncManager->EndFrame(Context.World);
	}

	Context.World = nullptr;
	LevelList.Reset();
}
```

## 설명
- [[UWorld/UWorld.Tick|UWorld::Tick]]에서 컬렉션 하나의 tick group들을 모두 돌린 뒤 호출되어, 한 프레임의 tick을 마무리한다.
- `bTickNewlySpawned = false`로 되돌려 tick phase가 끝났음을 표시한다. 이후 추가되는 tick function은 `NewlySpawnedTickFunctions`에 들어가지 않는다.
- `LevelList`의 모든 [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]에 대해 [[FTickTaskLevel/FTickTaskLevel.EndFrame|FTickTaskLevel::EndFrame()]]을 호출한다.
- 마지막에 `Context.World`를 nullptr로, `LevelList`를 비워 `StartFrame()`에서 채워둔 프레임 컨텍스트를 정리한다.
