# ⚙ 통합 이벤트 시스템 개발

라이브 게임 운영에 필요한 다양한 이벤트를 구조화하고 자동화하기 위해  
**기획 시트 개선, 공통 API 도입, MVC 기반 팝업 시스템** 등  
이벤트 파이프라인 전반을 설계하고 주도적으로 개선했습니다.

---

## 🔍 문제 정의

- 각 이벤트가 JSON 문자열을 직접 입력하는 `event_script` 단일 필드 기반으로 구성
- 이벤트마다 API/모델/처리 방식이 달라 유지보수 및 협업 비효율 초래
- 팝업과 컨트롤러에 로직/뷰가 혼재되어 분석과 수정이 어려움  
  무분별한 싱글톤 호출로 인한 스파게티 구조 발생

---

## 🛠 주요 개선 사항

### ✅ 1. 시트 구조 개선 (기획자 편의성 향상 및 휴먼 이슈 최소화)

- JSON 필드 기반 설정 → **컬럼 기반 정형화된 구조**로 변경
- 기획자 및 서버 개발자와 협업하여 **반복 가능하고 실수 방지에 유리한 시트 구조 제안 및 주도적 설계**
- 데이터 구조 통일로 자동 파싱 및 재사용 가능성 향상

| id | event_id | _event_name | category | key | value |
|----|----------|-------------|----------|-----|--------|
| 1  | 1        | 공룡 키우기 이벤트 | 공통     | 이벤트 종료 | 0 |
| 2  | 1        | 공룡 키우기 이벤트 | 보상체계 | 주간 보상   | 체력 아이템 |

<sub>※ 실제 사용된 데이터가 아닌 예시입니다.</sub>

---

### ✅ 2. 공통 이벤트 컨트롤러 구축 (클라이언트 구조 표준화)

- `BaseEventController<T>`를 제네릭 및 인터페이스 기반으로 추상화
- 이벤트마다 필요한 로직만 override하여 **독립적으로 구현 가능**
- **싱글톤 호출 남용 제거**, 상태 판단은 컨트롤러, UI는 View로 분리

```csharp
public interface IDinosaurEventController : IEventController
{
    void ShowDinosaur();
}

public class DinosaurEventController :
    BaseEventController<DinosaurEventController, DinosaurEventControllerData, IDinosaurEventController>
{
    // 전용 로직만 override
}
```
### ✅ 3. 서버 API 통합 및 처리 방식 개선
- 각 이벤트마다 달랐던 API를 /event/progress, /event/reward 등 공통 API로 통합
- 공통 모델을 받아 자동 갱신 → 이벤트마다 API/모델을 새로 만들 필요 없음

```json
// /event/progress 예시
{
  "1": {
    "general": {
      "closeEvent": "2025-06-01"
    },
    "rewards": {
      "today": "2025-06-01",
      "rewards": [...]
    }
  }
}
```
### ✅ 4. 팝업 시스템 자동화 (MVC 구조 적용)
#### 🎯 문제점
- 팝업에 로직과 UI 제어 코드 혼합
- 컨트롤러가 직접 UI 요소를 수정하며 코드 재사용이 어려움

#### ✅ 개선 방향
- 팝업 UI(View)는 모델만 받아 UI 표시 전용 역할로 분리
- 컨트롤러는 EventPopupModel만 구성해 넘김
- 팝업은 SetDataFromController(model)에서 데이터 바인딩만 수행

```csharp
// Controller
eventPopup.SetDataFromController(model);

// View
public void SetDataFromController(EventPopupModel model)
{
    titleText.text = model.Title;
    descText.text = model.Description;
    rewardIcons.Set(model.Rewards);
}
```

### 📈 개선 효과
| 항목        | 개선 전                   | 개선 후                     |
| --------- | ---------------------- | ------------------------ |
| 시트 설정 오류  | 자주 발생                  | 거의 없음 (정형화 + 자동화)        |
| 이벤트 개발 속도 | 평균 1.5차수 소요            | 평균 1차수 내 완료              |
| 팝업 처리 구조  | 이벤트별 별도 구현<br>직접 UI 수정 | 공통 팝업 자동화<br>모델 기반 UI 갱신 |
| 서버 연동 방식  | 이벤트마다 API/모델 별도 구현     | 공통 API 구조 도입 및 분기처리화     |


### 💬 담당 역할 및 기여
- 실 운영 중인 구조의 문제를 스스로 인지하고, 클라이언트 파트장, 서버 개발자, 기획자들에게 구조 개선 방향을 제안
- 시트 구조 회의 및 기획자의 사용성 고려한 구조 설계 주도
- 공통 컨트롤러 구조 및 MVC 기반 팝업 시스템 직접 설계 및 구현
- 시스템 표준화와 자동화를 통해 개발 효율성, 협업 품질, 유지보수성 크게 향상

<div align="center">
✨ 이 시스템은 현재도 Live 환경에서 유지보수 중이며
신규 이벤트 개발/폴리싱 속도 향상, 휴먼이슈 감소,
협업 효율성 향상 등에 실질적인 기여를 하고,
동료 클라이언트 개발자들의 니즈에 맞춰 지속적으로 업데이트하고 있습니다.

</div>
