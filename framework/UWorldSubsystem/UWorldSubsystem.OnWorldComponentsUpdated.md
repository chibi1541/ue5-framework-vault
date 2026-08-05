---
related:
  - "[[USubsystem/USubsystem|USubsystem]]"
  - "[[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]"
  - "[[UWorld/UWorld|UWorld]]"
tags:
  - WorldSubsystem_h
---

```cpp
/** called after world components (e.g. line batcher and all level components) have been updated */
virtual void OnWorldComponentsUpdated(UWorld& World) {}
```

## 설명
- [[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]의 마지막 단계에서, 등록된 모든 `UWorldSubsystem`에 대해 호출되는 통지 함수.
- 엔진 기본 구현은 비어 있다. WorldSubsystem을 커스텀할 때 **world initialize 타이밍(모든 컴포넌트 등록이 끝난 시점)에 실행하고 싶은 로직**을 여기에 override해서 넣으라는 목적의 훅이다.
- 이 시점이면 line batcher와 모든 level component의 등록이 끝난 상태이므로, 컴포넌트에 접근하는 초기화 코드를 안전하게 둘 수 있다.
