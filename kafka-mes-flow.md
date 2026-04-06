# Kafka MES 파이프라인 전체 플로우

## 정상 경로

```mermaid
flowchart LR
    MES([MES 시스템]) --> RAW["mes.raw\n12p · 7d"]
    RAW --> FILTER[Filter Service]
    FILTER --> FILTERED["mes.filtered\n6p · 3d"]
    FILTERED --> LLM[LLM Selector\nqwen3:8b]
    LLM --> SEL["mes.llm.selected\n6p · 3d"]
    SEL --> PARSER[Parser Service\n플러그인 실행]
    PARSER --> PARSED["mes.parsed\n6p · 3d"]
    PARSED --> STORE[Storage Service]
    STORE --> DB[(Database)]
```

## 실패 경로

```mermaid
flowchart LR
    FAIL([어느 단계든 실패]) --> RETRY["mes.retry\n3p · 7d"]
    RETRY --> RS[Retry Service\nExponential Backoff]

    RS -->|"1차: 0s\n2차: 30s\n3차: 5m"| REROUTE{failed_stage\n기반 재투입}
    REROUTE -->|llm_selector| FILTERED["mes.filtered"]
    REROUTE -->|parser_service| SEL["mes.llm.selected"]
    REROUTE -->|storage_service| PARSED["mes.parsed"]

    RS -->|"3회 초과"| DLQ["mes.dlq\n1p · 30d"]
    DLQ --> MON[DLQ Monitor]
    MON -->|100건 초과| ALERT([Slack 알림])
    MON -->|수동 재처리| RETRY
```

## 핵심 요약

| 구성 요소 | 역할 | 핵심 설정 |
|---|---|---|
| **Kafka 토픽** | 단계별 메시지 버퍼 | acks=all, idempotence=true |
| **Filter Service** | 불필요 이벤트 제거 | raw 대비 30~50% 감소 |
| **LLM Selector** | parser 이름 분류 | confidence threshold: 0.85 / 0.60 |
| **Parser Service** | subprocess 격리 플러그인 실행 | 핫로딩, timeout=10s |
| **Retry Service** | Exponential Backoff 재시도 | 최대 3회, 단계별 재투입 |
| **DLQ Monitor** | 최종 실패 감시 | 100건 초과 시 Slack 알림 |
