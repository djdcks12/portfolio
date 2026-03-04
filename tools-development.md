**Tech Keywords**
- Unity Editor Extension
- Workflow Automation
- Internal Tooling
- ML-Agents
- Team Productivity

# 내부 툴 개발 및 팀 기여

본 문서는 게임 기능 개발 외에,
팀 생산성과 업무 효율 개선을 목적으로 수행한 **내부 툴 개발 및 기술 공유 활동**을 정리한 자료입니다.
모든 내용은 실제로 수행한 작업을 기준으로 정리했습니다.

---

## 간트 차트 기반 일정 시각화 툴

### 배경

팀 업무가 점점 많아지고 병렬적으로 진행되면서,
각자의 작업 일정과 진행 상태를 한눈에 파악하기 어려운 문제가 있었습니다.
사내에 PM 직군이 없는 상태에서 구두 공유나 문서 기반 관리로는 
일정 충돌이나 병목을 파악하는 데 한계가 있었습니다.

### 작업 내용

* Node.js 기반 로컬 웹 서버 형태의 일정 관리 툴 제작
* 간트 차트 형태로 업무 일정 시각화
* 담당자, 작업 기간, 진행 상태를 기준으로 일정 관리 가능하도록 구성
* SQLite 기반 경량 데이터베이스를 사용해 로컬 환경에서 안정적으로 운영

### 결과

* 팀 전체 일정 흐름을 직관적으로 파악 가능
* 업무 중복 및 일정 충돌 인지 용이
* 일정 관리에 대한 커뮤니케이션 비용 감소

---

## 맵툴 개선 및 레벨 제작 지원

### 배경

레벨 콘텐츠 제작 과정에서,
맵 데이터 검증 및 반복 수정 작업에 많은 시간이 소요되고 있었습니다.
레벨 담당자들의 요청 사항이 누적되면서,
툴 차원의 개선이 필요하다고 판단했습니다.

### 작업 내용

* 맵 데이터 유효성 검증을 위한 통계 타입 체크 기능 추가
* 다수 맵에 대해 블록 색상을 일괄 변경할 수 있는 기능 구현
* 맵 데이터 병합 시 스테이지 순서가 보장되도록 오름차순 머지 로직 개선
* 기타 레벨 담당자 요청에 따른 맵툴 기능 개선 및 버그 수정 대응

### 결과

* 레벨 제작 및 수정 작업 속도 향상
* 맵 데이터 오류 사전 발견 가능
* 레벨 담당자의 반복 작업 감소

---

## 오토플레이 시스템 개발

### 배경

레벨 담당자들이 맵 밸런스 검증을 위해 동일 스테이지를 수십 회 반복 플레이해야 했고,
QA에서도 매 업데이트마다 수동으로 전체 스테이지를 테스트하는 데 많은 공수가 투입되고 있었습니다.
이를 자동화하기 위해 오토플레이 시스템을 설계하고, 이후 ML 기반 확장까지 시도했습니다.

### 작업 내용

* **휴리스틱 기반 오토플레이**: 보드 상태를 분석하여 미션 목표, 특수 블록 생성 가능성, 매칭 범위 등을 종합적으로 고려한 우선순위 기반 최적 매칭 판단 로직 설계 및 구현
* **에디터 통합**: 에디터 상에서 자동 플레이 실행 및 결과 통계 확인 가능하도록 구성
* **ML-Agents 기반 확장 연구**: Unity ML-Agents를 활용하여 강화학습 기반 자동 플레이 에이전트 학습 시도, 휴리스틱 대비 다양한 전략 학습 가능성 확인
* **지속적 RND**: 랜덤 시드 고정 기반 재현 가능한 시뮬레이션 구축 및 고속 시뮬레이션 가능 여부 등 연구 진행

### 결과

* 레벨 담당자 반복 플레이 검증 업무 자동화, QA 수동 테스트 공수 대폭 감소
* ML-Agents 확장 연구 결과를 사내 세미나에서 발표 → 수상 (아래 세미나 참조)

<details>
<summary><b>구조 — 다중 팩터 우선순위 스코어링 시스템</b></summary>

#### 설계 의도

오토플레이의 핵심은 "지금 보드에서 어디를 먼저 터뜨려야 하는가"를 판단하는 것입니다.
단일 기준(예: 미션 목표만)으로는 복합적인 보드 상황을 판단할 수 없으므로,
**5개 카테고리의 점수를 셀 단위로 합산**하여 최적 타겟을 결정하는 구조를 설계했습니다.

```csharp
// --- 다중 팩터 우선순위 계산 흐름 (구조 예시) ---

// 1단계: 선처리 — 제외/0점 대상 사전 수집 + 예외 처리 계산
foreach (var index in AllValidCells())
{
    CollectExceptions(index);       // 타격 불가능한 셀 수집
    CollectZeroScoreTargets(index);  // 이미 목표 달성된 셀
    CollectCustomBonuses(index);     // 여러 예외 처리
}

// 2단계: 카테고리별 점수 합산
foreach (var index in AllValidCells())
{
    int block    = CalcBlockScore(index);     // 블록 타입·상태별 기본 점수
    int obstacle = CalcObstacleScore(index);  // 장애물 제거 우선순위
    int board    = CalcBoardScore(index);     // 보드 타일(얼음, 리본 등)

    // 어느 하나라도 "제외" 판정이면 해당 셀은 건너뜀
    if (IsExcluded(block, obstacle, board))
        continue;

    scoreMap[index] = block + obstacle + board;
}

// 3단계: 맥락적 보너스 적용 + 음수 클램핑
foreach (var key in scoreMap.Keys)
{
    scoreMap[key] += customBonusMap.GetValueOrDefault(key, 0);
    scoreMap[key] = Math.Max(0, scoreMap[key]);
}
```

#### 맥락적 보너스: 기본 카테고리로 표현할 수 없는 상황 판단

```csharp
// --- 맥락적 보너스 규칙 (구조 예시) ---
// 기본 점수만으로는 맥락적 판단이 불가능. 
// 이를 별도 카테고리로 분리하고, 각 규칙의 조건과 점수를 시트 데이터로 관리하여
// 코드 수정 없이 판단 기준을 추가·조정할 수 있도록 설계.
{
    foreach (var rule in contextualRules)
    {
        switch (rule.type)
        {
            case A:
            case B:
            case C:

            // 예외처리 데이터 관리
        }
    }
}
```

이 구조 덕분에 시트 데이터에 조작만으로 오토플레이 판단 기준을 조정할 수 있었습니다.

#### 우선순위 맵을 활용한 최적 매칭 판단

위에서 계산된 우선순위 맵을 오토플레이가 실제로 사용하는 흐름입니다.
모든 매칭 가능한 패턴을 탐색하고, **효과 범위 내 우선순위 합산이 가장 높은 수를 선택**합니다.

```csharp
// --- 오토플레이 최적 수 선택 (구조 예시) ---

// 우선순위 맵 생성 (위 코드의 결과물)
var priorityScoreMap = CalculateTotalScores();

// Phase 1: 일반 매칭 — 한 인덱스 당 가능 한 모든 패턴 × 전체 셀에서 최고 점수 탐색
foreach (var pattern in matchPatterns)
{
    foreach (var cell in allValidCells)
    {
        // 패턴 매칭 가능 여부 확인 (같은 색, 장애물 미차단 등)
        if (!CanMatch(pattern, cell)) continue;

        var matchedCells = GetMatchedCells(pattern, cell);

        // 패턴 기본 점수 (매칭 패턴에 따른 보정 점수(ex 특수 블록등이 만들어지면 점수가 높음))
        int score = GetPatternBaseScore(pattern);

        // 보드판 우선순위 상황에 따른 점수
        score += priorityScoreMap.Socre(matchedCells);

        if (score > bestResult.maxScore)
            bestResult = new Result(score, matchedCells);
    }
}

// Phase 2: 특수블록 — 효과 범위 내 우선순위 비교
// 각 특수블록의 효과 범위를 계산하고 범위 내 우선순위를 합산하여 비교
foreach (var specialBlock in allSpecialBlocks)
{
    var effectArea = CalcEffectArea(specialBlock);  // 타입별 효과 범위 계산
    int score = priorityScoreMap.Score(effectArea);

    if (score > bestResult.maxScore)
        bestResult = new Result(score, specialBlock);
}
```

이 2단계 구조(우선순위 맵 생성 → 매칭 판단에 활용) 덕분에,
판단 기준과 매칭 로직이 분리되어 각각 독립적으로 확장할 수 있었습니다.

</details>

---

## 사내 개발 세미나 참여 및 발표

### 개요

사내 개발 문화 활성화를 목적으로 진행된 개발 세미나에 참여하여,
직접 경험하거나 학습한 내용을 공유했습니다.

### 발표 주제

1. **매치3 퍼즐 게임 자동플레이 – 밸런싱 자동화와 ML 도전**

   * 위 오토플레이 시스템의 구현 과정과 ML-Agents 확장 연구를 주제로 발표
2. **Unity Sprite Atlas v2와 AssetBundle 연동 실험기**

   * Sprite Atlas v2 실제 적용 과정에서의 실험 및 검증 결과 공유
3. **선형대수학으로 설명하는 트랜스포머 구조 개요**

   * 개인 학습 내용을 기반으로 한 AI/ML 기초 개념 공유

### 결과

* 사내 지식 공유 및 기술 논의 활성화에 기여
* 세미나 초기 참여자로서 발표 문화 정착에 기여
* 사내 개발 세미나 상 수상

---

## 정리

툴 개발과 지식 공유 활동은,
단순한 편의 기능 추가가 아니라 **팀/회사 전체의 작업 흐름을 개선하기 위한 시도**였습니다.

이러한 경험을 통해,
문제를 개인 작업으로만 해결하기보다
**구조와 도구를 통해 반복 비용을 줄이는 접근 방식**을 중요하게 생각하게 되었습니다.
