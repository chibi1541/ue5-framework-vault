---
related:
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase|FSubsystemCollectionBase]]"
  - "[[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeSubsystem|FSubsystemCollectionBase::AddAndInitializeSubsystem]]"
tags:
  - SubsystemCollection_h
---

```cpp
struct FSubsystemCollectionInitialization
{
	// Classes of subsystem we intend to initialize, for handling calls to GetSubsystem during initialization
	// Key is a base/interface/concrete class which may be used in a call to GetSubsystem
	// Value is the concrete class that will be returned
	// So when key == value, this is a concrete class we intend to initialize
	TMap<UClass*, UClass*> ClassMap;

	// List of classes to be initialized. Classes may be initialized before we reach them in the queue via
	// re-entrancy.
	// We also may add more classes to the queue from re-entrancy, e.g. loading modules.
	TArray<UClass*> Queue;
};
```

## 설명
- [[FSubsystemCollectionBase/FSubsystemCollectionBase.AddAndInitializeSubsystem|FSubsystemCollectionBase::AddAndInitializeSubsystem]] 내부에서 지역 변수(`LocalInitialization`)로 사용되며, 초기화 도중 재진입(re-entrant) 되는 `GetSubsystem` 호출을 처리하기 위한 대기열/매핑을 담는다.
- `ClassMap`은 key==value면 실제로 초기화할 concrete 클래스, 그 외에는 상속/인터페이스 탐색용 매핑. `Queue`는 초기화 대상 클래스 목록.
