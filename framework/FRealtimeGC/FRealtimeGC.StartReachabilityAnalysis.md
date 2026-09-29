---
related:
  - "[[FRealtimeGC/FRealtimeGC.PerformReachabilityAnalysis|FRealtimeGC::PerformReachabilityAnalysis]]"
  - "[[FRealtimeGC/FRealtimeGC.MarkObjectsAsUnreachable|FRealtimeGC::MarkObjectsAsUnreachable]]"
tags:
  - GarbageCollection_cpp
---

```cpp
void StartReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	{
		const double StartTime = FPlatformTime::Seconds();
		MarkObjectsAsUnreachable(KeepFlags);
		const double ElapsedTime = FPlatformTime::Seconds() - StartTime;
		if (!Stats.bFoundGarbageRef)
		{
			GGCStats.MarkObjectsAsUnreachableTime = ElapsedTime;
		}
		UE_LOGF(LogGarbage, Verbose, "%f ms for MarkObjectsAsUnreachable Phase (%d Objects To Serialize)", ElapsedTime * 1000, InitialObjects.Num());
	}
}
```

## 설명
- reachability analysis의 시작 단계로 [[FRealtimeGC/FRealtimeGC.MarkObjectsAsUnreachable|MarkObjectsAsUnreachable()]]를 호출한다.
- 초기 pass(`!Stats.bFoundGarbageRef`)일 때만 소요 시간을 `GGCStats.MarkObjectsAsUnreachableTime`에 기록한다.
