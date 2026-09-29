---
related:
  - "[[FRealtimeGC/FRealtimeGC.MarkObjectsAsUnreachable|FRealtimeGC::MarkObjectsAsUnreachable]]"
  - "[[FGCFlags/FGCFlags.SetMaybeUnreachable_ForGC|FGCFlags::SetMaybeUnreachable_ForGC]]"
tags:
  - GarbageCollection_cpp
---

```cpp
void ResetReachabilityFlags(const EGatherOptions Options)
{
	// Options가 EGatherOptions::Parallel인 경우 병렬 처리를 위한 준비
	using FMarkObjectsState = TThreadedGather<TArray<UObject*>>;
	FMarkObjectsState MarkObjectsState;

	// 병렬 처리를 위한 쓰레드 모음
	MarkObjectsState.Start(Options, GUObjectArray.GetObjectArrayNum(), GUObjectArray.GetFirstGCIndex());

	// Reset all objects to 'MaybeUnreachable' state
	FMarkObjectsState::FThreadIterators& ThreadIterators = MarkObjectsState.GetThreadIterators();
	ParallelFor(TEXT("GC.ResetReachabilityFlags"), MarkObjectsState.NumWorkerThreads(), 1, [&ThreadIterators](int32 ThreadIndex)
		{
			FMarkObjectsState::FIterator& ThreadState = ThreadIterators[ThreadIndex];
			for (; ThreadState.Index <= ThreadState.LastIndex; ++ThreadState.Index)
			{
				FUObjectItem* ObjectItem = &GUObjectArray.GetObjectItemArrayUnsafe()[ThreadState.Index];
				if (ObjectItem->GetObject())
				{
					FGCFlags::SetMaybeUnreachable_ForGC(ObjectItem);
				}
			}
		}, (MarkObjectsState.NumWorkerThreads() == 1) ? EParallelForFlags::ForceSingleThread : EParallelForFlags::None);

	TArray<UObject*> Unused; // We didn't collect any objects when resetting so this is just a dummy to satisfy the API
	MarkObjectsState.Finish(Unused);

}
```

## 설명
- `GUObjectArray`의 GC 대상 구간(`GetFirstGCIndex()` 이후)을 `TThreadedGather`로 thread별 구간으로 나누고, `ParallelFor`로 병렬 순회한다.
- 유효한 object마다 [[FGCFlags/FGCFlags.SetMaybeUnreachable_ForGC|FGCFlags::SetMaybeUnreachable_ForGC()]]를 호출해 `MaybeUnreachable` 상태로 되돌린다.
- worker thread가 1개면 `EParallelForFlags::ForceSingleThread`로 실행한다.
- 수집하는 object는 없으므로 `Finish()`에는 dummy 배열을 넘긴다.
