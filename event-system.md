**Tech Keywords**
- Data-driven Event System
- MVC Architecture
- State Management
- Live Ops Framework

# 통합 이벤트 시스템 설계

## 개요

라이브 서비스 퍼즐 게임에서 이벤트 콘텐츠 수가 증가하면서,
이벤트 구현 방식의 불일치로 인한 유지보수 비용과 운영 리스크가 지속적으로 발생했습니다.
이를 해결하기 위해 이벤트를 개별 기능 단위가 아닌 **공통 구조를 가진 시스템**으로 정리하고,
기획·서버·클라이언트가 동일한 규칙 안에서 작업할 수 있도록 통합 이벤트 시스템을 설계·구현했습니다.

---

## 문제 상황

이벤트가 누적되며 다음과 같은 문제가 반복되었습니다.

* 이벤트마다 UI 및 로직 구조가 달라 유지보수가 어려움
* 기획 시트 포맷이 제각각이라 해석 오류 및 휴먼 이슈 발생
* 서버 API 구조 차이로 같은 동작이라도 클라이언트 코드를 매번 새로 구현
* 이벤트 폴리싱 때 마다 새로 만드는 수준으로 공수

문제의 핵심은 이벤트를 **일회성 콘텐츠 단위로 구현**하고,
장기 운영을 고려한 구조적 기준이 없었던 점이라고 판단했습니다.

---

## 설계 목표

* 이벤트를 코드가 아닌 **데이터 중심 구조**로 정의
* 신규 이벤트 추가 시 기존 구조를 재사용할 수 있도록 표준화
* UI, 로직, 데이터 흐름의 역할 분리

---

## 시스템 구조

### Event Definition

* 모든 이벤트를 공통 포맷의 정의 데이터로 관리
* 이벤트 ID, 기간, 조건, 보상, UI 타입을 데이터로 분리

```text
EventDefinition
- EventId
- EventType
- StartTime / EndTime
- ConditionSet
- RewardSet
- UiType
```

이벤트의 성격과 동작을 코드가 아닌 데이터로 판단하도록 구성했습니다.

<details>
<summary><b>핵심 구조 — 이벤트 컨트롤러 ↔ 팝업 갱신 흐름</b></summary>

#### 제네릭 베이스 컨트롤러

이벤트마다 데이터 모델과 UI는 다르지만, **생명주기는 동일**합니다.
공통 흐름을 제네릭 베이스에 고정하고, 이벤트별 차이는 타입 파라미터로 분리했습니다.

```
  BaseEventController<TData, TInterface, ...>
  ┌─────────────────────────────────────────────────┐
  │  Active           — 서버 시간 기준 활성 판정     │
  │  Data (TData)     — 서버 모델에서 자동 바인딩    │
  │  ShowPopup<T>()   — 팝업 생성 + 데이터 주입      │
  │  RequestReward()  — 보상 요청 → 상태 갱신        │
  │  RefreshStatus()  — 모든 열린 팝업 일괄 갱신     │
  └────────────────────────┬────────────────────────┘
                           │ 상속
  ┌────────────────────────┴────────────────────────┐
  │  구체 이벤트 컨트롤러                              │
  │  - EventType, BundleName만 지정                  │
  │  - OnRequestReward() 등 고유 로직만 override      │ 
  │  - 추가되는 API만 따로 구현                        │      
  └─────────────────────────────────────────────────┘
```

#### 팝업 생성 → 데이터 주입 → 타이머 등록

팝업은 컨트롤러를 통해서만 열립니다.
열리는 순간 컨트롤러의 인터페이스를 주입하고, 미션 상태를 바인딩합니다.

```csharp
// --- 팝업 생성 흐름 (구조 예시) ---
LoadFromBundle(popupType, bundleName, (popup) =>
{
    // 팝업에 컨트롤러 인터페이스 주입
    // → 팝업은 인터페이스를 통해서만 컨트롤러에 접근
    popup.Initialize(controllerInterface);

    // 현재 미션/보상 데이터를 팝업에 바인딩
    SetDataPopup(popup);

    // Rx 타이머: 1초 간격으로 잔여 시간 갱신
    // → 자정 경과 시 오늘의 미션 목록 갱신
    // → 이벤트 종료 감지 시 팝업 자동 닫기
    Observable.Interval(TimeSpan.FromSeconds(1))
        .Subscribe(_ =>
        {
            popup.UpdateEventTime(leftTime);
            if (leftTime.TotalSeconds < 1) ClosePopup(popup);
        })
        .AddTo(popup.gameObject);  // 팝업 파괴 시 구독 자동 해제
});
```

#### 상태 변경 → 열려있는 모든 팝업 일괄 갱신

보상 수령, 미션 증가, 광고 시청 등 **모든 상태 변경의 끝에는 동일한 갱신 흐름**이 실행됩니다.

```csharp
// --- 상태 변경 → 전체 팝업 갱신 (구조 예시) ---

// 1. 서버 모델 기준으로 미션 진행 상태 재계산
RecalculateMissionProgress();

// 2. 열려 있는 모든 팝업에 갱신된 데이터 주입
foreach (var popup in openPopups)
{
    // 공통 데이터 바인딩 (미션 목록 + 보상 목록)
    popup.SetDataFromController(rewardDict, missionStatus);
    // 이벤트별 추가 데이터는 override로 확장
    SetAdditionalDataPopup(popup);
}

// 3. 월드맵 아이콘도 갱신
SetCheckUI();
```

#### 보상 요청 → 갱신 흐름 전체

```
유저 보상 버튼 터치
    │
    ▼
RequestReward(mission)
    │  서버에 보상 요청
    ▼
서버 응답 (성공)
    │
    ├─ RefreshMissionStatus()     ← 상태 재계산 + 열린 팝업 전체 갱신
    │
    ├─ OnRequestReward(rewards)   ← 이벤트별 후처리 (override)
    │
    └─ 보상 획득 팝업 노출
```

</details>

---

### Event Controller

* 이벤트 상태 관리는 공통 EventController에서 처리
* 활성 여부, 서버 동기화, 보상 요청 흐름을 표준화
* 이벤트별 구현은 최소화하고 공통 흐름 위주로 설계

```text
IEventController
- Initialize()
- RefreshState()
- RequestReward()
```

이를 통해 이벤트별 분기 코드를 줄이고,
유지보수와 디버깅 비용을 낮추는 것을 목표로 했습니다.

---

### UI 구조

* 이벤트 UI는 MVC 구조를 기준으로 통일
* View는 표현만 담당하고 로직과 상태 관리는 Controller에 위임
* 팝업 및 페이지 UI를 공통 베이스 구조로 관리

UI 변경이 이벤트 로직에 영향을 주지 않도록 설계했습니다.

---

## 데이터 흐름

1. 정형화된 기획 시트 기반 이벤트 데이터 작성
2. 데이터 변환 후 클라이언트 로드
3. EventController 초기화 및 상태 반영

운영 중 이벤트 수치 조정 시 코드 수정 없이 데이터만으로 대응할 수 있도록 구성했습니다.

---

## 협업 측면에서의 변화

* 기획: 이벤트 구조에 대한 이해 비용 감소, 시트 수정 중심 작업 가능
* 서버: 이벤트 API 구조 통일로 재사용성 향상
* 클라이언트: 신규 이벤트 개발 및 유지보수 시간 감소

---

## 결과

* 이벤트 평균 개발 소요 기간 단축 — 공통 구조 재사용으로 **데이터 연동 약 1주 + UI 작업 약 1주 수준으로 병렬 진행 가능**한 구조 확보
* 기획 시트 포맷 표준화로 해석 오류 및 휴먼 이슈 발생 건수 감소
* 이벤트 구현 방식에 대한 팀 내 기준 정립, 신규 인원 온보딩 시 학습 비용 절감

---

## 회고 및 구조적 한계

이 시스템을 실제 서비스에 적용해 운영하면서 몇 가지 한계도 분명히 인식하게 되었습니다.

초기 구조는 View가 Controller에 강하게 종속되는 형태였기 때문에, 해당 구조에 대한 사전 이해가 없으면 학습에 시간이 필요했고 UI 구성 방식에도 여러 제약이 존재했습니다.

또한 이벤트 통합이 진행될수록 Base가 되는 EventController가 비대해지고, 이벤트별 분기 처리로 인해 init이나 상태 처리 등의 복잡도가 증가하는 문제도 발생했습니다.

더불어 당시에는 "View는 아무것도 해서는 안 된다"는 관념에 강하게 의존한 나머지, 구조적으로 View를 강하게 통제하는 방향으로 설계가 이루어졌습니다. View는 Controller의 존재를 알지 못한 채, Controller가 내려주는 정보만을 정해진 타이밍에 맞춰 자동으로 갱신되는 구조였고, 이로 인해 UI 확장이나 예외적인 화면 흐름을 처리할 때 유연성이 떨어지는 한계가 드러났습니다.

<details>
<summary><b>코드로 보는 구조적 한계</b></summary>

#### 현재 구조의 제약

```csharp
// 컨트롤러가 팝업에 데이터를 밀어넣는 구조
// 팝업은 이 메서드가 호출될 때, 내려준 데이터로만 갱신 가능
popup.SetDataFromController(rewardDict, missionStatus);
SetAdditionalDataPopup(popup);  // 추가 데이터가 필요하면 이걸 override


class EventPopup : BaseEventPopup                                                
{                                                                                
    // 이 메서드가 호출될 때만 데이터가 갱신됨                                   
    // 팝업이 스스로 필요한 데이터를 가져올 수 없음                              
    public void SetDataFromController(rewards, missions)                                   
    {                                                                                                          
        // rewards로 리워드 관련 UI 갱신                                               
        // missions로 미션 관련 UI 갱신                                           
    }
    public void SetDataFromController(eventModel)                                   
    {                                                                            
        // eventModel로 관련 UI 갱신                                         
    }                                                                     
                                                                            
    // 만약 다른 시스템의 데이터가 필요하다면?                                   
    // → 컨트롤러의 eventModel에 해당 내용 계속 추가.                        
    //   컨트롤러에서 다른 컨트롤러 이곳 저곳 접근하며 거기서 내려줘야 함 
    // → 컨트롤러가 비대해지는 원인                         
}                  
```

#### 고려하고 있는 방향

팝업이 컨트롤러에 종속되지 않고, 열리면서 스스로 필요한 데이터를 찾아가는 구조도
가능하지 않을까 하는 생각을 하고 있습니다.

```csharp
// 팝업이 열리면서 필요한 데이터를 직접 조회
// 데이터를 조작하지는 않되, 읽는 것은 자유롭게
class EventPopup : BasePopup
{
    void OnOpen()
    {
        var missions = MissionManager.Instance.GetCurrentMissions(eventId);
        var rewards = RewardManager.Instance.GetRewards(eventId);
        var ranking = RankingManager.Instance.GetMyRank();
        // 컨트롤러 수정 없이 팝업에서 필요한 데이터를 확장 가능
    }
}
```

상태 변경(보상 요청 등)은 여전히 컨트롤러를 통하되,
데이터 조회는 팝업이 직접 수행하는 형태입니다.
이렇게 하면 컨트롤러가 오로지 데이터를 조작만하고 Popup에 대한 통제는 없음
로직, 모델과 View가 더 분리되어 스파게티가 될 요소가 덜함.

</details>

이 경험을 통해 이후에는 다음과 같은 구조도 충분히 고려할 수 있겠다는 판단을 하게 되었습니다.

* 이벤트별로 개별 Singleton 기반 이벤트 클래스를 분리
* 기존에는 공통 기능을 Base 클래스 상속으로 처리했으나, 이로 인해 초기화 순서, 상태 갱신 타이밍, 의존 관계가 복잡해지는 문제가 발생
* 이후에는 상속 중심 구조보다는 공통 기능을 명확한 책임 단위의 서비스/유틸로 분리하고, 이벤트 클래스는 필요한 기능만 조합해 사용하는 방식이 더 적절할 수 있다고 판단
* 팝업(View)이 직접 데이터를 조작하지는 않되, 자신에게 필요한 데이터는 해당 책임을 가진 Singleton 객체에 직접 접근해 조회하는 방식

즉, 강한 중앙 통제형 구조가 아닌 느슨한 결합 구조가 장기 라이브 서비스 환경에서는 더 유연할 수 있다는 관점을 얻게 되었고, 이후 시스템 설계 시 중요한 기준 중 하나로 삼고 있습니다.
