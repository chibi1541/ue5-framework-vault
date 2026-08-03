---
related:
  - "[[ULevel/ULevel|ULevel]]"
  - "[[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]"
  - "[[ULevel/ULevel.IncrementalUpdateComponents|ULevel::IncrementalUpdateComponents]]"
tags:
  - Level_cpp
---

```cpp
/** update all components of actors associated with this level (aka in Actors array) and creates the BSP model components */
void UpdateLevelComponents(bool bRerunConstructionScripts, FRegisterComponentContext* Context = nullptr)
{
    // update all components in one swoop
    // 첫번째 인자에 0을 넣으면 증분적으로 Update하는 것이 아닌 한번에 싹다 업데이트 해버림
    // bRerunConstructionScripts : BP 편집 후 컴파일 버튼으로 변경 내용을 기반으로 다시 CS를 호출하는 옵션
    // 이미 쿠킹을 진행해서 asset을 최적화(빌드)한 상태라면 해당 옵션은 false?
    IncrementalUpdateComponents(0, bRerunConstructionScripts, Context);
}
```

## 설명
- [[UWorld/UWorld.UpdateWorldComponents|UWorld::UpdateWorldComponents]]가 각 레벨마다 호출하는 함수. `Actors` 배열에 들어있는 액터들의 component를 전부 등록한다.
- 실제 작업은 [[ULevel/ULevel.IncrementalUpdateComponents|IncrementalUpdateComponents()]]에 위임하며, 첫 인자 `NumComponentsToUpdate`에 **0**을 넘긴다. 0은 "증분적으로 나눠서"가 아니라 "한 번에 전부 업데이트"를 의미한다.
- `bRerunConstructionScripts`는 BP를 편집하고 컴파일했을 때 변경 내용을 기준으로 ConstructionScript를 다시 호출할지에 대한 옵션이다. 이미 쿠킹되어 asset이 빌드된 상태라면 다시 돌릴 이유가 없다.
