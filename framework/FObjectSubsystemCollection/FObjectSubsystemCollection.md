---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection.ForEachSubsystem|FObjectSubsystemCollection::ForEachSubsystem]]"
  - "[[FObjectSubsystemCollection/FObjectSubsystemCollection.GetSubsystemArrayCopy|FObjectSubsystemCollection::GetSubsystemArrayCopy]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[USubsystem/USubsystem|USubsystem]]"
tags:
  - SubsystemCollection_h
---

```cpp
// World의 서브시스템 컬렉션 변수
FWorldSubsystemCollection SubsystemCollection;

class FWorldSubsystemCollection : public FObjectSubsystemCollection<UWorldSubsystem>
{
	
}

template<typename TBaseType>
class FObjectSubsystemCollection : public FSubsystemCollectionBase
{
		

}
```

## 설명
- [[UWorld/UWorld|UWorld]]가 들고 있는 서브시스템 컨테이너 변수 `SubsystemCollection`의 타입 계층.
- `FWorldSubsystemCollection` → `FObjectSubsystemCollection<UWorldSubsystem>` → [[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]] 순으로 상속된다.
- `FObjectSubsystemCollection<TBaseType>`은 타입 안전성만 얹는 얇은 템플릿 레이어다. 실제 데이터(`SubsystemMap`, `SubsystemArrayMap`)와 로직은 전부 non-template인 `FSubsystemCollectionBase`에 있다. 템플릿을 base까지 끌고 가면 코드가 타입마다 중복 생성되므로, 타입 검사만 템플릿 쪽에서 하고 구현은 base로 내리는 구조다.
- `FWorldSubsystemCollection`은 `TBaseType`을 `UWorldSubsystem`으로 고정하기 위한 alias 성격의 클래스다.
