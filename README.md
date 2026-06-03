# US Treasury Rate-Hike Timing Harness

미국채선물에 반영된 **금리(정책 전환) 시점**을 4개 전문가 에이전트 팀으로 분석하는 하네스입니다.

## 구성

```
.claude/
├── agents/                          # 전문가 에이전트 정의 (누가)
│   ├── interest-rate-analyst.md     # 금리 전문가 — 정책금리 경로·시점
│   ├── economy-analyst.md           # 거시/미시 경제 전문가 — 펀더멘털(원인)
│   ├── bond-analyst.md              # 채권 전문가 — 시장 내재 시점
│   └── fx-analyst.md                # 환율 전문가 — 대외 제약·피드백
└── skills/                          # 분석 방법론 (어떻게)
    ├── rate-policy-analysis/        # 금리 분석법
    ├── macro-micro-economy-analysis/# 경제 분석법
    ├── bond-market-analysis/        # 채권 분석법
    ├── fx-impact-analysis/          # 환율 분석법
    └── treasury-rate-hike-orchestrator/  # 오케스트레이터 (조율)
```

## 실행 모드: 에이전트 팀
금리·경제·채권·환율은 상호의존적이므로, 팀원 간 `SendMessage` 교차 토론으로 **정책 의도 시점 vs 시장 내재 시점의 괴리**를 포착하는 것이 핵심입니다.

## 사용법
이 디렉토리에서 Claude Code를 실행하고, 예를 들어:
> "미국채선물 기준으로 다음 금리 인하 시점을 분석해줘"

라고 요청하면 오케스트레이터가 팀을 구성해 분석을 수행하고 `미국채선물_금리시점_분석보고서.md`를 생성합니다.

## 면책
모든 산출물은 정보 제공·교육 목적의 분석이며 투자 권유·금융 자문이 아닙니다.
