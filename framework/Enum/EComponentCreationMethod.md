---
related:
  - "[[UActorComponent/UActorComponent.IsCreatedByConstructionScript|UActorComponent::IsCreatedByConstructionScript]]"
tags:
  - ComponentInstanceDataCache_h
---

```cpp
UENUM()
enum class EComponentCreationMethod : uint8
{
	/** A component that is part of a native class. */
	Native,
	/** A component that is created from a template defined in the Components section of the Blueprint. */
	SimpleConstructionScript,
	/**A dynamically created component, either from the UserConstructionScript or from a Add Component node in a Blueprint event graph. */
	UserConstructionScript,
	/** A component added to a single Actor instance via the Component section of the Actor's details panel. */
	Instance,
};
```

## 설명
- `UActorComponent`가 **어떤 경로로 생성되었는지**를 기술하는 enum. `UActorComponent::CreationMethod`에 저장된다.
- `Native` ⇒ C++ 코드(native 클래스)에 의해 생성됨.
- `SimpleConstructionScript` ⇒ BP의 Hierarchy(Components) 창에서 추가된 템플릿으로부터 생성됨.
- `UserConstructionScript` ⇒ UserConstructionScript 함수(BP의 ConstructionScript 이벤트)나 event graph의 Add Component 노드에서 동적으로 생성됨.
- `Instance` ⇒ Actor의 Detail 패널에서 개별 인스턴스에 추가된 컴포넌트.
- [[UActorComponent/UActorComponent.IsCreatedByConstructionScript|IsCreatedByConstructionScript()]]는 이 중 `SimpleConstructionScript`와 `UserConstructionScript`를 묶어서 판별한다.
