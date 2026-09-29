---
related:
  - "[[FRealtimeGC/FRealtimeGC.StartReachabilityAnalysis|FRealtimeGC::StartReachabilityAnalysis]]"
  - "[[FRealtimeGC/FRealtimeGC.ResetReachabilityFlags|FRealtimeGC::ResetReachabilityFlags]]"
  - "[[FRealtimeGC/FRealtimeGC.MarkRootObjectsAsReachable|FRealtimeGC::MarkRootObjectsAsReachable]]"
tags:
  - GarbageCollection_cpp
---

```cpp
FORCENOINLINE void MarkObjectsAsUnreachable(const EObjectFlags KeepFlags)
{
	// EGatherOptions::None, EGatherOptions::Parallel
	EGatherOptions GatherOptions = GetObjectGatherOptions();

	// Don't swap the flags if we're re-entering this function to track garbage references
	if (const bool bInitialMark = !Stats.bFoundGarbageRef)
	{
		// This marks all UObjects as MaybeUnreachable
		FGCFlags::SwapReachableAndMaybeUnreachable();
	}
	else
	{
		// Swapping flags would inverse reachability results from the initial (normal) pass but what we want
		// is to reset reachability state of all objects to 'MaybeUnreachable'
		ResetReachabilityFlags(GatherOptions);
	}
	
	// Now make sure all clustered objects and root objects are marked as Reachable. 
	// This could be considered as initial part of reachability analysis and could be made incremental.
	MarkClusteredObjectsAsReachable(GatherOptions, InitialObjects);
	MarkRootObjectsAsReachable(GatherOptions, KeepFlags, InitialObjects);
}
```

## 설명
- 모든 UObject를 `MaybeUnreachable` 상태로 만든 뒤, cluster와 root object만 다시 `Reachable`로 marking한다.
- 초기 pass에서는 `FGCFlags::SwapReachableAndMaybeUnreachable()`로 flag 의미 자체를 swap해 모든 object를 한 번에 `MaybeUnreachable`로 만든다(object를 순회하지 않음).
- garbage tracking rerun(`Stats.bFoundGarbageRef`)에서는 swap하면 앞선 pass 결과가 뒤집히므로, [[FRealtimeGC/FRealtimeGC.ResetReachabilityFlags|ResetReachabilityFlags()]]로 모든 object를 직접 순회하며 `MaybeUnreachable`로 되돌린다.
- `MarkClusteredObjectsAsReachable()`, [[FRealtimeGC/FRealtimeGC.MarkRootObjectsAsReachable|MarkRootObjectsAsReachable()]]의 결과는 `InitialObjects`에 모여 이후 참조 탐색의 시작점이 된다.
- `EGatherOptions`: `None`(단일 thread) 또는 `Parallel`.
