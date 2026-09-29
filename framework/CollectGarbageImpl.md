---
related:
  - "[[CollectGarbageIncremental]]"
  - "[[CollectGarbageFull]]"
  - "[[FRealtimeGC/FRealtimeGC.PerformReachabilityAnalysis|FRealtimeGC::PerformReachabilityAnalysis]]"
tags:
  - GarbageCollection_cpp
---

```cpp
template<bool bPerformFullPurge>
AUTORTFM_DISABLE void CollectGarbageImpl(EObjectFlags KeepFlags)
{
	{
		// Reachability analysis.
		{
			const EGCOptions Options = GetReferenceCollectorOptions(bPerformFullPurge);

			// Perform reachability analysis.
			FRealtimeGC GC;
			GC.PerformReachabilityAnalysis(KeepFlags, Options);
		}
	}
}
```

## 설명
- `bPerformFullPurge`를 template 인자로 받아 `GetReferenceCollectorOptions()`로 `EGCOptions`를 만든다.
- `FRealtimeGC` 인스턴스를 생성해 [[FRealtimeGC/FRealtimeGC.PerformReachabilityAnalysis|FRealtimeGC::PerformReachabilityAnalysis()]]로 reachability analysis를 수행한다.
