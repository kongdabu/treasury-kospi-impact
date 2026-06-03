---
name: treasury-rate-hike-orchestrator
description: "미국채선물 금리 인상/인하(정책 전환) 시점을 금리·경제·채권·환율 4개 전문가 에이전트 팀으로 분석하는 오케스트레이터. 미국채선물 금리 시점, 연준 금리 인상/인하 시점, FOMC 전망, 금리 전환 분석을 요청하면 반드시 이 스킬을 사용. 후속 작업: 분석 다시 실행, 업데이트, 재실행, 특정 전문가 영역만 수정·보완, 이전 결과 기반 개선, 최신 데이터로 갱신 요청 시에도 반드시 사용."
---

# Treasury Rate-Hike Timing Orchestrator

미국채선물에 반영된 금리(정책 전환) 시점을 4개 전문가 에이전트 팀으로 분석하여, 확률 가중 시점 컨센서스 보고서를 생성하는 통합 스킬.

## 실행 모드: 에이전트 팀
4명의 분석 영역(금리·경제·채권·환율)은 강하게 상호의존한다(예: 경제→금리, 금리→채권, 채권→환율, 환율→금리). 따라서 팀원 간 `SendMessage` 교차 토론이 결과 품질의 핵심이다. 단순 병렬 수집(서브 에이전트)으로는 괴리·피드백을 포착할 수 없다.

## 아키텍처: 팬아웃/팬인 + 토론

## 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 스킬 | 출력 |
|------|-------------|------|------|------|
| interest-rate-analyst | general-purpose | 정책금리 경로·시점 | rate-policy-analysis | `_workspace/02_interest-rate-analyst_policy-path.md` |
| economy-analyst | general-purpose | 펀더멘털(원인) | macro-micro-economy-analysis | `_workspace/02_economy-analyst_fundamentals.md` |
| bond-analyst | general-purpose | 시장 내재 시점 | bond-market-analysis | `_workspace/02_bond-analyst_market-signal.md` |
| fx-analyst | general-purpose | 대외 제약·피드백 | fx-impact-analysis | `_workspace/02_fx-analyst_fx-impact.md` |
| (리더 = 오케스트레이터) | — | 통합 보고서 | — | `미국채선물_금리시점_분석보고서.md` |

> 모든 팀원은 `general-purpose` 빌트인 타입 + `model: "opus"`. 웹 리서치(WebSearch/WebFetch)로 최신 데이터를 확보해야 하므로 읽기 전용 타입(Explore)은 쓰지 않는다.

## 워크플로우

### Phase 0: 컨텍스트 확인 (후속 작업 지원)
1. 작업 디렉토리에서 `_workspace/` 존재 여부 확인
2. 분기:
   - **미존재** → 초기 실행. Phase 1로
   - **존재 + 부분 수정 요청**(예: "환율 분석만 다시") → 부분 재실행. 해당 팀원만 단독 호출(서브 에이전트), 그 산출물만 갱신 후 Phase 4 재통합
   - **존재 + 새 입력/기준일** → 새 실행. 기존 `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동 후 Phase 1

### Phase 1: 준비
1. 사용자 입력 분석 — 분석 대상(금리 인상/인하 전환), **분석 기준일**(미지정 시 오늘), 관심 시계(예: 향후 12개월) 파악
2. `_workspace/` 생성, 사용자 제공 자료가 있으면 `_workspace/00_input/`에 저장
3. 공통 전제(분석 기준일, 시계, 출력 포맷)를 팀 프롬프트에 명시할 준비

### Phase 2: 팀 구성
```
TeamCreate(
  team_name: "treasury-rate-team",
  members: [
    { name: "interest-rate-analyst", agent_type: "general-purpose", model: "opus",
      prompt: "rate-policy-analysis 스킬로 정책금리 경로·시점을 분석. 기준일 {date}, 시계 {horizon}. 출력 파일 저장 후 팀과 교차 검증." },
    { name: "economy-analyst", agent_type: "general-purpose", model: "opus",
      prompt: "macro-micro-economy-analysis 스킬로 펀더멘털 트리거 조건을 진단. ..." },
    { name: "bond-analyst", agent_type: "general-purpose", model: "opus",
      prompt: "bond-market-analysis 스킬로 시장 내재 시점을 역산. ..." },
    { name: "fx-analyst", agent_type: "general-purpose", model: "opus",
      prompt: "fx-impact-analysis 스킬로 환율↔금리 피드백을 분석. ..." }
  ]
)
TaskCreate(tasks: [
  { title: "펀더멘털 트리거 진단", assignee: "economy-analyst" },
  { title: "정책금리 경로·시점 추정", assignee: "interest-rate-analyst", depends_on: ["펀더멘털 트리거 진단"] },
  { title: "시장 내재 시점 역산·괴리 분석", assignee: "bond-analyst", depends_on: ["정책금리 경로·시점 추정"] },
  { title: "환율 피드백·대외 변수 진단", assignee: "fx-analyst", depends_on: ["시장 내재 시점 역산·괴리 분석"] },
  { title: "교차 토론·전제 정합화", assignee: "interest-rate-analyst" }
])
```
> 의존성은 "이상적 정보 흐름" 순서(경제→금리→채권→환율)일 뿐, 팀원은 병렬로 작업하며 SendMessage로 수시 조율한다. 엄격한 직렬 대기를 강제하지 않는다.

### Phase 3: 분석 + 교차 토론
**실행 방식:** 팀원 자체 조율 (에이전트 팀)

각 팀원은 자기 스킬로 분석하고 표준 시나리오 표를 산출한다. 핵심은 아래 통신이다:

```
economy ──→ interest-rate   인플레·고용이 시사하는 정책 방향
interest-rate ──→ bond       정책금리 경로 (선물 내재 경로와 대조 요청)
bond ──→ fx                  금리차·수익률 변화 (달러 영향)
fx ──→ interest-rate         달러·금융여건이 정책에 주는 제약(피드백)
모든 팀원 ──→ 공유 작업 목록   TaskUpdate로 진행률
```

**상충 처리:** 정책 의도 시점(금리)과 시장 내재 시점(채권)이 괴리되면, 삭제·평균내지 말고 SendMessage로 토론한 뒤 **양측을 출처와 함께 병기**한다. 이 괴리가 보고서의 핵심 인사이트다.

**리더 모니터링:** TaskGet으로 진행률 확인, 막힌 팀원에게 SendMessage로 개입.

### Phase 4: 통합 (팬인)
1. 모든 작업 완료 대기 (TaskGet)
2. 4개 산출물 Read
3. 통합 로직:
   - 4개 시나리오 표를 **하나의 종합 시점 매트릭스**로 합성
   - 베이스/상방/하방별로 4개 관점의 확률·전제를 종합해 **확률 가중 컨센서스 시점** 도출
   - **정책 vs 시장 괴리**, **대외 제약**을 별도 섹션으로 강조
   - 상충 데이터는 출처 병기, 미해소 불확실성을 명시
4. 최종 보고서 생성: 작업 디렉토리에 `미국채선물_금리시점_분석보고서.md`
   - 구성: ① 요약(컨센서스 시점·확률) ② 시점 매트릭스 ③ 4개 영역별 핵심 ④ 정책-시장 괴리 ⑤ 대외 리스크 ⑥ 모니터링 지표·일정 ⑦ 데이터 출처·기준일 ⑧ **면책 조항**

### Phase 5: 정리
1. 팀원에게 종료 요청 (SendMessage)
2. 팀 정리 (TeamDelete)
3. `_workspace/` 보존 (사후 검증·감사 추적용)
4. 사용자에게 결과 요약 보고 + 피드백 요청

## 데이터 흐름
```
economy ─┐
         ├─ SendMessage 교차 토론 → 각자 02_*.md 저장
interest─┤
bond ────┤
fx ──────┘
         └──── 리더 Read 4파일 → 종합 시점 매트릭스 → 최종 보고서
```

## 데이터 전달 프로토콜
- 태스크 기반(진행 조율) + 파일 기반(산출물 `_workspace/02_*.md`) + 메시지 기반(실시간 토론) 조합
- 파일명: `_workspace/{phase}_{agent}_{artifact}.md`, 최종 보고서만 작업 디렉토리 루트에 출력

## 에러 핸들링
| 상황 | 전략 |
|------|------|
| 팀원 1명 실패/중지 | 리더 감지 → SendMessage 상태 확인 → 재시작, 실패 시 해당 영역 누락 명시하고 진행 |
| 팀원 과반 실패 | 사용자에게 알리고 진행 여부 확인 |
| 최신 데이터 접근 불가 | 확보 가능한 최근 데이터 기준으로 분석, 기준일·한계를 보고서에 명시 |
| 시점 추정 상충 | 삭제·평균 금지. 출처 병기 + 괴리 섹션에 분석 |
| 타임아웃 | 부분 결과로 Phase 4 진행, 미완료 영역 명시 |

## 테스트 시나리오
### 정상 흐름
1. 사용자: "미국채선물 기준 다음 금리 인하 시점 분석해줘"
2. Phase 1: 기준일=오늘, 시계=12개월 확정, `_workspace/` 생성
3. Phase 2: 4명 팀 + 5개 작업 등록
4. Phase 3: 각자 분석 + SendMessage 교차 토론(정책↔시장 괴리 도출)
5. Phase 4: 4개 산출물 통합 → 확률 가중 컨센서스 시점 보고서
6. Phase 5: 팀 정리, `_workspace/` 보존
7. 예상 결과: `미국채선물_금리시점_분석보고서.md` 생성(면책 포함)

### 에러 흐름
1. Phase 3에서 fx-analyst가 환율 데이터 접근 실패로 중지
2. 리더가 유휴 알림 수신 → SendMessage 상태 확인 → 1회 재시작
3. 재실패 시 fx 영역을 "대외 변수 일부 미수집"으로 명시하고 나머지 3개로 Phase 4 진행
4. 최종 보고서 ⑤ 대외 리스크 섹션에 한계 기록

## 면책
최종 보고서는 정보 제공·교육 목적의 분석이며 투자 권유·금융 자문이 아니다. 보고서 말미에 반드시 명시한다.
