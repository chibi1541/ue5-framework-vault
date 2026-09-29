---
related:
  - "[[FRealtimeGC/FRealtimeGC.ResetReachabilityFlags|FRealtimeGC::ResetReachabilityFlags]]"
tags:
  - GarbageCollectionInternalFlags_h
---

```cpp
FORCEINLINE static void SetMaybeUnreachable_ForGC(FUObjectItem* ObjectItem)
{
	// EInternalObjectFlags::ReachabililtyFlag0 삭제
	ObjectItem->AtomicallyClearFlag_ForGC(ReachableObjectFlag);
	// EInternalObjectFlags::ReachabilityFlag1 설정
	// 이 상태가 아직 GC 추적이 안된 상태? 이 상태를 유지하면 Purge 대상이 되는 듯?
	ObjectItem->AtomicallySetFlag_ForGC(MaybeUnreachableObjectFlag);
}
```

## 설명
- `FUObjectItem`의 internal flag를 atomic하게 변경해 object를 `MaybeUnreachable` 상태로 만든다.
  - `ReachableObjectFlag`(`EInternalObjectFlags::ReachabilityFlag0`) 제거
  - `MaybeUnreachableObjectFlag`(`EInternalObjectFlags::ReachabilityFlag1`) 설정
- reachability analysis가 끝날 때까지 이 상태로 남은 object는 unreachable로 판정되어 purge 대상이 되는 것으로 보인다.
