---
related:
  - "[[UObjectBase/UObjectBase|UObjectBase]]"
  - "[[UObjectBaseUtility/UObjectBaseUtility.IsTemplate|UObjectBaseUtility::IsTemplate]]"
  - "[[FRealtimeGC/FRealtimeGC.MarkRootObjectsAsReachable|FRealtimeGC::MarkRootObjectsAsReachable]]"
tags:
  - ObjectMacros_h
---

```cpp
/** flags describing an object instance */
// [ ] see the unreal code
enum EObjectFlags
{
    RF_Transactional			=0x00000008,	///< Object is transactional.
    RF_ClassDefaultObject		=0x00000010,	///< This object is used as the default template for all instances of a class. One object is created for each class
		RF_ArchetypeObject			=0x00000020,	///< This object can be used as a template for instancing objects. This is set on all types of object templates
};
```

## 설명
- 객체 인스턴스의 속성을 기술하는 비트 플래그. [[UObjectBase/UObjectBase|UObjectBase]]의 `ObjectFlags`에 저장된다.
- `RF_Transactional`: transactional 객체(에디터의 undo/redo 대상). `UWorld::CreateWorld`에서 새 월드에 설정한다.
- `RF_ClassDefaultObject`: 클래스당 하나씩 생성되는 CDO. `RF_ArchetypeObject`: 인스턴싱의 template으로 쓸 수 있는 객체.
