---
related:
  - "[[FTickTaskManager/FTickTaskManager.AddTickFunction|FTickTaskManager::AddTickFunction]]"
  - "[[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]"
  - "[[ULevel/ULevel|ULevel]]"
tags:
  - TickTaskManager_cpp
---

```cpp
FTickTaskLevel* TickTaskLevelForLevel(ULevel* Level, bool bCreateIfNeeded = true)
{
	check(Level);

	if (bCreateIfNeeded && Level->TickTaskLevel == nullptr)
	{
		// 레벨에 TickTaskLevel이 없는 경우 여기서 할당
		// AllocateTickTaskLevel() => return new FTickTaskLevel;
		Level->TickTaskLevel = AllocateTickTaskLevel();
	}

	check(Level->TickTaskLevel);
	return Level->TickTaskLevel;
}
```

## 설명
- [[ULevel/ULevel|ULevel]]에 대응하는 [[FTickTaskLevel/FTickTaskLevel|FTickTaskLevel]]을 돌려준다. 실체는 `ULevel::TickTaskLevel` 멤버를 꺼내오는 것뿐이다.
- `bCreateIfNeeded`가 true(기본값)이고 아직 할당되지 않았다면 `AllocateTickTaskLevel()`(= `new FTickTaskLevel`)로 여기서 할당한다. 보통은 `ULevel` 생성자에서 이미 할당되어 있다.
