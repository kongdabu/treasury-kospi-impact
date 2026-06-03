# US Treasury Harness (금리 시점 + 국내 증시 영향)

## 하네스 A: 미국채선물 금리 시점 분석

**목표:** 미국채선물에 반영된 금리(정책 전환) 시점을 금리·경제·채권·환율 4개 전문가 에이전트 팀으로 분석하여, 확률 가중 시점 컨센서스 보고서를 생성한다.

**트리거:** 미국채선물 금리 시점 / 연준 금리 인상·인하 시점 / FOMC 전망 / 금리 전환 '시점'이 핵심인 작업을 요청하면 `treasury-rate-hike-orchestrator` 스킬을 사용하라. 단순 용어 질문은 직접 응답 가능.

## 하네스 B: 미국채선물 → 국내 증시 영향도 분석

**목표:** 미국채 선물 동향이 환율·유가·지정학을 거쳐 국내 증시(KOSPI)로 전이되는 영향도를 채권·환율·지정학유가·국내증시 4개 전문가 에이전트 팀으로 분석하여, 지수 시나리오·업종 차별화·종목 쏠림 리스크 보고서를 생성한다.

**트리거:** 미국채 선물의 국내 증시 영향 / 미국 금리·환율·유가·미이란 전쟁의 KOSPI 충격 / 외국인 수급 / 종목·섹터 쏠림 관련 작업을 요청하면 `treasury-kospi-impact-orchestrator` 스킬을 사용하라. (금리 전환 '시점' 자체가 핵심이면 하네스 A.)

**구성:** 두 하네스 모두 에이전트 팀 모드(팬아웃/팬인 + 토론). bond·fx 분석가는 두 하네스가 공유한다. 상세는 각 오케스트레이터 `.claude/skills/*/SKILL.md` 및 `.claude/agents/` 참조.

**면책:** 모든 산출물은 정보 제공·교육 목적이며 투자 권유가 아니다.

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-06-01 | 초기 구성 (금리·경제·채권·환율 4인 팀 + 오케스트레이터) | 전체 | - |
| 2026-06-03 | 하네스 B 추가 (지정학유가·국내증시 2인 + treasury-kospi-impact-orchestrator). bond·fx 재사용 | agents/geopolitics-oil-analyst, agents/kospi-impact-analyst, skills/geopolitics-oil-analysis, skills/kospi-transmission-analysis, skills/treasury-kospi-impact-orchestrator | 미국채 선물의 국내 증시 전이(미이란·유가·환율·쏠림) 분석 요청 |
