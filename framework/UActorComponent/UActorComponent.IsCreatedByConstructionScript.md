---
related:
  - "[[UActorComponent/UActorComponent.RegisterComponentWithWorld|UActorComponent::RegisterComponentWithWorld]]"
  - "[[Enum/EComponentCreationMethod|EComponentCreationMethod]]"
tags:
  - ActorComponent_cpp
---

```cpp
// ActorComponent의 멤버 변수
EComponentCreationMethod CreationMethod;

/** returns true if instances of this component are created by either the user or simple construction script */
bool IsCreatedByConstructionScript() const
{
    return ((CreationMethod == EComponentCreationMethod::SimpleConstructionScript) || (CreationMethod == EComponentCreationMethod::UserConstructionScript));
}
```

## 설명
- 컴포넌트가 construction script(SCS/CCS)로 생성된 인스턴스인지 판별한다.
- 판별 기준은 멤버 변수 `CreationMethod`([[Enum/EComponentCreationMethod|EComponentCreationMethod]])가 `SimpleConstructionScript` 또는 `UserConstructionScript`인지 여부다.
- [[UActorComponent/UActorComponent.RegisterComponentWithWorld|RegisterComponentWithWorld()]]에서 이 값이 true면, outer가 자신인 자식 컴포넌트들이 등록에서 누락될 수 있으므로 재귀적으로 register를 수행한다.
