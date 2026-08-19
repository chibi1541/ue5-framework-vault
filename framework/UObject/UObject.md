---
related:
  - "[[UObjectBaseUtility/UObjectBaseUtility|UObjectBaseUtility]]"
  - "[[UWorld/UWorld|UWorld]]"
  - "[[ULevel/ULevel|ULevel]]"
  - "[[AActor/AActor|AActor]]"
  - "[[FTickPrerequisite/FTickPrerequisite|FTickPrerequisite]]"
tags:
  - Object_h
---

```cpp
/**
 * the base class of all UE objects. the type of an object is defined by its UClass
 * this provides support functions for creating and using objects, and virtual functions that should be overriden in child classes
 */
 class UObject : public UObjectBaseUtility
{
    
};
```

## 설명
- 모든 UE 객체의 base class. 객체의 타입은 `UClass`로 정의된다.
- 상속 구조는 `UObject` → [[UObjectBaseUtility/UObjectBaseUtility|UObjectBaseUtility]] → [[UObjectBase/UObjectBase|UObjectBase]] 순이며, 주로 볼 내용은 `UObjectBase`의 `ObjectFlags`와 `OuterPrivate`다.
