---
name: treasury-kospi-impact-orchestrator
description: "미국채 선물 동향이 환율·유가·지정학을 거쳐 국내 증시(KOSPI)로 전이되는 영향도를 채권·환율·지정학유가·국내증시 4개 전문가 에이전트 팀으로 분석하는 오케스트레이터. 미국채 선물 국내 증시 영향, 미국 금리·환율·유가·미이란 전쟁의 KOSPI 충격, 외국인 수급, 종목/섹터 쏠림 분석을 요청하면 반드시 이 스킬을 사용. 후속 작업: 분석 다시 실행, 업데이트, 재실행, 특정 전문가 영역만(환율만·유가만·국내증시만) 수정·보완, 이전 결과 기반 개선, 최신 데이터로 갱신 요청 시에도 반드시 사용. (연준 금리 전환 '시점' 자체가 핵심이면 treasury-rate-hike-orchestrator를 사용.)"
---

# Treasury → KOSPI Impact Orchestrator

미국채 선물 동향이 환율·유가·지정학을 거쳐 **국내 증시로 전이되는 영향도**를 4개 전문가 에이전트 팀으로 분석하여, 지수 시나리오·업종 차별화·종목 쏠림 리스크를 담은 통합 영향분석 보고서를 생성하는 스킬.

> 이 오케스트레이터는 기존 `treasury-rate-hike-orchestrator`(연준 금리 '시점' 분석)와 짝을 이룬다. 같은 repo의 bond·fx 분석가를 재사용하되, 초점이 **국내 증시 전이**라는 점이 다르다.

## 실행 모드: 에이전트 팀
4개 영역(채권·환율·지정학유가·국내증시)은 단방향이 아니라 상호의존한다(금리→환율→외국인수급, 지정학→유가→인플레→금리, 위험회피→안전자산→증시). kospi-impact-analyst는 상류 3개를 종합하는 팬인 지점이므로, `SendMessage` 교차 토론과 파일 공유가 결과 품질의 핵심이다. 단순 병렬 수집으로는 전이의 정합성을 보장할 수 없다.

## 아키텍처: 팬아웃(상류 3) → 팬인(국내증시) + 토론

## 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 스킬 | 출력 |
|------|-------------|------|------|------|
| bond-analyst | general-purpose | 미국채 선물·수익률 동향 | bond-market-analysis | `_workspace/02_bond-analyst_market-signal.md` |
| fx-analyst | general-purpose | 환율·달러·자본흐름 | fx-impact-analysis | `_workspace/02_fx-analyst_fx-impact.md` |
| geopolitics-oil-analyst | general-purpose | 미이란 전쟁·유가·안전자산 | geopolitics-oil-analysis | `_workspace/02_geopolitics-oil-analyst_geo-oil.md` |
| kospi-impact-analyst | general-purpose | 국내 증시 전이·쏠림(팬인) | kospi-transmission-analysis | `_workspace/03_kospi-impact-analyst_transmission.md` |
| (리더 = 오케스트레이터) | — | 통합 보고서 | — | `미국채선물_국내증시_영향분석보고서.md` |

> 모든 팀원은 `general-purpose` 빌트인 타입 + `model: "opus"`. 최신 데이터를 WebSearch/WebFetch로 확보해야 하므로 읽기 전용(Explore)은 쓰지 않는다.

## 워크플로우

### Phase 0: 컨텍스트 확인 (후속 작업 지원)
1. 작업 디렉토리에서 `_workspace/` 존재 여부 확인
2. 분기:
   - **미존재** → 초기 실행. Phase 1로
   - **존재 + 부분 수정 요청**(예: "유가 분석만 다시", "국내 증시 영향만 갱신") → 부분 재실행. 해당 팀원만 단독 호출(서브 에이전트), 그 산출물만 갱신 후 Phase 4 재통합. 단 kospi-impact-analyst를 갱신하면 상류 변경 여부를 확인.
   - **존재 + 새 입력/기준일** → 새 실행. 기존 `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동 후 Phase 1

### Phase 1: 준비
1. 사용자 입력 분석 — **분석 기준일**(미지정 시 오늘), 관심 시계(예: 향후 3~6개월), 강조 채널(외국인 수급/쏠림/유가 등) 파악
2. `_workspace/` 생성, 사용자 제공 자료가 있으면 `_workspace/00_input/`에 저장
3. 공통 전제(분석 기준일, 시계, 출력 포맷)를 팀 프롬프트에 명시할 준비

### Phase 2: 팀 구성
```
TeamCreate(
  team_name: "treasury-kospi-team",
  members: [
    { name: "bond-analyst", agent_type: "general-purpose", model: "opus",
      prompt: "bond-market-analysis 스킬로 미국채 선물·수익률·곡선 동향을 분석. 기준일 {date}, 시계 {horizon}. 출력 저장 후 팀과 교차 검증. 국내증시 전이를 위해 금리 경로를 kospi-impact-analyst에게 전달." },
    { name: "fx-analyst", agent_type: "general-purpose", model: "opus",
      prompt: "fx-impact-analysis 스킬로 달러·원/달러 환율·자본흐름을 분석. 외국인 수급 트리거가 되는 환율 경로를 kospi-impact-analyst에게 전달(핵심). ..." },
    { name: "geopolitics-oil-analyst", agent_type: "general-purpose", model: "opus",
      prompt: "geopolitics-oil-analysis 스킬로 미이란 전쟁·유가·안전자산 선호를 분석. 유가 민감 업종·risk-off 영향을 kospi-impact-analyst에게 전달. ..." },
    { name: "kospi-impact-analyst", agent_type: "general-purpose", model: "opus",
      prompt: "kospi-transmission-analysis 스킬로 상류 3개 산출물을 종합해 KOSPI 전이·외국인수급·종목쏠림·업종차별화를 도출. 상류 산출물(02_*.md)을 Read하고 SendMessage 신호를 반영. ..." }
  ]
)
TaskCreate(tasks: [
  { title: "미국채 선물·수익률 동향 분석", assignee: "bond-analyst" },
  { title: "환율·달러·외국인 자본흐름 분석", assignee: "fx-analyst" },
  { title: "미이란 전쟁·유가·안전자산 충격 분석", assignee: "geopolitics-oil-analyst" },
  { title: "국내 증시 전이·쏠림 종합(팬인)", assignee: "kospi-impact-analyst",
    depends_on: ["미국채 선물·수익률 동향 분석", "환율·달러·외국인 자본흐름 분석", "미이란 전쟁·유가·안전자산 충격 분석"] },
  { title: "교차 토론·전이 정합화", assignee: "kospi-impact-analyst" }
])
```
> 상류 3명(bond·fx·geopolitics-oil)은 병렬로 작업하며 SendMessage로 수시 조율한다. kospi-impact-analyst는 상류 산출이 어느 정도 나오면 착수하되, 추가 신호를 받아 갱신한다. 엄격한 직렬 대기는 강제하지 않는다.

### Phase 3: 분석 + 교차 토론
**실행 방식:** 팀원 자체 조율 (에이전트 팀)

각 상류 팀원은 자기 스킬로 분석하고 표준 시나리오 표를 산출한다. 핵심 통신:
```
geopolitics-oil ──→ bond        안전자산 선호·인플레 재점화가 미국채에 주는 양방향 충격
geopolitics-oil ──→ fx          위험회피의 달러·원화·산유국 통화 영향
bond ──→ fx                     금리차·수익률 변화 (달러 영향)
bond ──→ kospi-impact           금리 경로 → 성장주 밸류에이션·외국인 채권자금
fx ──→ kospi-impact             원/달러 → 외국인 주식 수급·수출주 (가장 강한 연계)
geopolitics-oil ──→ kospi-impact 유가 민감 업종·risk-off 자금 이탈
모든 팀원 ──→ 공유 작업 목록      TaskUpdate로 진행률
```

**상충 처리:** 상류 신호가 충돌하면(예: 금리↑인데 위험선호↑로 증시↑) 삭제·평균내지 말고 SendMessage로 토론한 뒤 **양측을 출처와 함께 병기**한다. 전이의 비선형성·괴리가 핵심 인사이트다.

**리더 모니터링:** TaskGet으로 진행률 확인, 막힌 팀원에게 SendMessage로 개입. kospi-impact-analyst가 상류 입력을 제대로 소비하는지 점검.

### Phase 4: 통합 (팬인)
1. 모든 작업 완료 대기 (TaskGet)
2. 4개 산출물 Read (상류 3 + 전이 종합 1)
3. 통합 로직:
   - kospi-impact-analyst의 **지수 시나리오 표**와 **업종 차별화 맵**을 보고서의 중심에 둔다
   - 상류 3개 영역(채권·환율·지정학유가)의 핵심을 "전이 원인" 섹션으로 정리
   - **외국인 수급·환율 연계**, **종목/섹터 쏠림 리스크**, **안전자산·금리 밸류에이션**을 별도 강조 섹션으로
   - 상충 데이터는 출처 병기, 미해소 불확실성·꼬리 위험을 명시
4. 최종 보고서 생성: 작업 디렉토리에 `미국채선물_국내증시_영향분석보고서.md`
   - 구성: ① 요약(지수 방향·핵심 리스크) ② 지수 시나리오 매트릭스 ③ 업종 차별화 맵 ④ 전이 원인(채권·환율·지정학유가) ⑤ 외국인 수급·환율 연계 ⑥ 종목/섹터 쏠림 리스크 ⑦ 안전자산·금리 밸류에이션 ⑧ 모니터링 지표·일정 ⑨ 데이터 출처·기준일 ⑩ **면책 조항**

### Phase 5: 정리
1. 팀원에게 종료 요청 (SendMessage)
2. 팀 정리 (TeamDelete)
3. `_workspace/` 보존 (사후 검증·감사 추적용)
4. 사용자에게 결과 요약 보고 + 피드백 요청

## 데이터 흐름
```
geopolitics-oil ─┐
                 ├─ SendMessage 교차 토론 → 각자 02_*.md 저장
bond ────────────┤              │
fx ──────────────┘              ▼
                        kospi-impact-analyst (Read 02_*.md 3개 + 신호)
                                 │  → 03_*.md (지수 시나리오 + 업종 맵)
                                 ▼
                 리더 Read 4파일 → 통합 영향분석 보고서
```

## 데이터 전달 프로토콜
- 태스크 기반(진행 조율) + 파일 기반(산출물 `02_*.md`/`03_*.md`) + 메시지 기반(실시간 토론) 조합
- 파일명: `_workspace/{phase}_{agent}_{artifact}.md`, 최종 보고서만 작업 디렉토리 루트에 출력
- kospi-impact-analyst는 상류 `02_*.md`를 반드시 Read한 뒤 `03_*.md`를 생성한다(팬인 보장)

## 에러 핸들링
| 상황 | 전략 |
|------|------|
| 상류 팀원 1명 실패/중지 | 리더 감지 → SendMessage 상태 확인 → 1회 재시작, 실패 시 해당 채널 누락 명시. kospi-impact는 확보 채널만으로 전이 진행 |
| kospi-impact-analyst 실패 | 핵심 팬인이므로 우선 재시작. 재실패 시 상류 3개 요약만으로 리더가 전이를 정성적으로 통합하고 한계 명시 |
| 팀원 과반 실패 | 사용자에게 알리고 진행 여부 확인 |
| 최신 데이터 접근 불가 | 확보 가능한 최근 데이터 기준으로 분석, 기준일·한계를 보고서에 명시 |
| 상류 신호 상충 | 삭제·평균 금지. 출처 병기 + 괴리/비선형 섹션에 분석 |
| 타임아웃 | 부분 결과로 Phase 4 진행, 미완료 영역 명시 |

## 테스트 시나리오
### 정상 흐름
1. 사용자: "미국채 선물 동향이 국내 증시에 주는 영향 분석해줘. 미이란 전쟁·유가·환율·종목 쏠림 다 고려해서"
2. Phase 1: 기준일=오늘, 시계=3~6개월 확정, `_workspace/` 생성
3. Phase 2: 4명 팀(bond·fx·geopolitics-oil·kospi-impact) + 5개 작업 등록
4. Phase 3: 상류 3명 병렬 분석 + SendMessage 교차 토론 → kospi-impact 팬인 종합
5. Phase 4: 4개 산출물 통합 → 지수 시나리오·업종 차별화·쏠림 리스크 보고서
6. Phase 5: 팀 정리, `_workspace/` 보존
7. 예상 결과: `미국채선물_국내증시_영향분석보고서.md` 생성(면책 포함)

### 에러 흐름
1. Phase 3에서 geopolitics-oil-analyst가 유가 데이터 접근 실패로 중지
2. 리더가 유휴 알림 수신 → SendMessage 상태 확인 → 1회 재시작
3. 재실패 시 유가 채널을 "일부 미수집"으로 명시하고, kospi-impact는 채권·환율 2개 채널로 전이 진행
4. 최종 보고서 ⑦ 안전자산·밸류에이션 및 한계 섹션에 기록

## 면책
최종 보고서는 정보 제공·교육 목적의 분석이며 투자 권유·금융 자문이 아니다. 특정 종목·업종 언급은 전이 경로 설명용 예시다. 보고서 말미에 반드시 명시한다.
