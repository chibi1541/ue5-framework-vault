---
related:
  - "[[CollectGarbageInternal]]"
tags:
  - GarbageCollection_cpp
---

```cpp
bool TryCollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	if (AutoRTFM::IsTransactional())
	{
		// Memory cannot be freed within a transaction as this would prevent us from rolling back to the initial state.
		UE_LOGF(LogGarbage, Log, "TryCollectGarbage: skipping garbage collection because an AutoRTFM transaction is active.");
		return false;
	}

	// No other thread may be performing UObject operations while we're running so try to acquire GC lock
	if (UE::GC::GIsIncrementalReachabilityPending)
	{
		// Since we're already in the middle of a previous GC acquire GC lock even if it means we have to block main thread
		AcquireGCLock();
	}	
	else if (!FGCCSyncObject::Get().TryGCLock())
	{
		if (GNumRetriesBeforeForcingGC > 0 && GNumAttemptsSinceLastGC > GNumRetriesBeforeForcingGC)
		{
			// Force acquire GC lock and block main thread		
			UE_LOGF(LogGarbage, Warning, "TryCollectGarbage: forcing GC after %d skipped attempts.", GNumAttemptsSinceLastGC);
			GNumAttemptsSinceLastGC = 0;
			AcquireGCLock();
		}
		else
		{
			++GNumAttemptsSinceLastGC;
			return false;
		}
	}

	// Perform actual garbage collection
	UE::GC::CollectGarbageInternal(KeepFlags, bPerformFullPurge);

	// GC lock was released after reachability analysis inside CollectGarbageInternal

	return true;

}
```

## 설명
- GarbageCollection chapter의 진입점. GC lock 획득을 시도하고, 성공하면 [[CollectGarbageInternal]]로 실제 GC를 수행한다.
- AutoRTFM transaction이 활성화된 상태에서는 rollback이 불가능해지므로 GC를 건너뛰고 `false`를 반환한다.
- 이미 incremental reachability analysis가 진행 중(`GIsIncrementalReachabilityPending`)이면 main thread를 block하더라도 `AcquireGCLock()`으로 lock을 강제로 잡는다.
- `TryGCLock()`에 실패하면 `GNumAttemptsSinceLastGC`를 증가시키고 포기한다. 실패 횟수가 `GNumRetriesBeforeForcingGC`를 넘으면 lock을 강제로 획득해 GC를 진행한다.
- `KeepFlags`: 이 flag를 가진 object는 GC 대상에서 제외된다. `bPerformFullPurge`: incremental이 아닌 full purge 여부.
