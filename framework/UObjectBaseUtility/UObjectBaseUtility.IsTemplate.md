---
related:
  - "[[UObjectBaseUtility/UObjectBaseUtility|UObjectBaseUtility]]"
  - "[[Enum/EObjectFlags|EObjectFlags]]"
  - "[[UActorComponent/UActorComponent.SetComponentTickEnabled|UActorComponent::SetComponentTickEnabled]]"
  - "[[UActorComponent/UActorComponent.SetupActorComponentTickFunction|UActorComponent::SetupActorComponentTickFunction]]"
tags:
  - UObjectBaseUtility_cpp
---

```cpp
/** determine whether this object is a template object */
// haker: 
// - I have been look through the unreal engine source code for long time, but I still can't explain what is archetype object with specific example
// - you just think of it as CDO, class default object
//   - for CDO, what I understand is like **initialization list** as default UObject instance
bool IsTemplate(EObjectFlags TemplateTypes = RF_ArchetypeObject|RF_ClassDefaultObject) const
{
    // haker: note that if one of outer is template, the object is template
    for (const UObjectBaseUtility* TestOuter = this; TestOuter; TestOuter = TestOuter->GetOuter())
    {
        if (TestOuter->HasAnyFlags(TemplateTypes))
            return true;
    }
    return false;
}
```

## 설명
- 실제로 인스턴스화된 객체가 아니라 템플릿화된 더미(archetype/CDO)인지를 판별한다.
- 판정 플래그는 기본값으로 `RF_ArchetypeObject | RF_ClassDefaultObject`([[Enum/EObjectFlags|EObjectFlags]])를 쓴다. CDO는 기본적으로 `RF_ArchetypeObject`를 함께 갖는다.
- CDO만 archetype인 것은 아니다. BP 클래스의 컴포넌트 템플릿도 `RF_ArchetypeObject`를 갖지만 CDO는 아니다.
- 자기 자신부터 outer chain을 따라 `ULevel`까지 올라가며 검사하고, **하나라도** template 플래그가 걸리면 template으로 판정한다. 반대로 CDO가 들고 있는 sub-object가 전부 CDO 플래그를 갖는 것은 아니기 때문에 이런 체크가 필요하다.
