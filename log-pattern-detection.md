# 정적 로그 패턴 탐지 시스템 — 실전 설계

---

## 1. 전체 최종 아키텍처

```mermaid
flowchart TD
    LOG([Log File]) --> CHUNK[Chunking]
    CHUNK --> DRAIN[Template Parsing\nDrain]

    DRAIN --> FEAT[Feature 생성]
    FEAT --> SEQ[Sequence 생성]
    FEAT --> FREQ[Frequency 통계]
    FEAT --> EMBED[Embedding]

    SEQ --> PSTORE[Pattern Store\n빈도 / 시퀀스]
    FREQ --> PSTORE
    EMBED --> VDB[Vector DB]
    DRAIN --> RAW[Raw Storage]

    subgraph QUERY[Query 처리]
        QIN([사용자 질의]) --> LLM_P[LLM Planner\n선택]
        LLM_P --> HYBRID[Hybrid Search]
        HYBRID --> VS[Vector Search]
        HYBRID --> CF[조건 필터]
        HYBRID --> PQ[Pattern 조회]
    end

    VDB --> VS
    PSTORE --> PQ
    RAW --> CF

    VS --> ENGINE[분석 엔진]
    CF --> ENGINE
    PQ --> ENGINE

    ENGINE --> SM[Sequence Mining]
    ENGINE --> CL[Clustering]
    ENGINE --> GR[Graph 분석]

    SM --> OUT([결과 출력])
    CL --> OUT
    GR --> OUT
```

---

## 2. 핵심 설계 포인트

### ① 입력 처리 (가장 중요)

```
Log → Chunk → Template 추출
```

> 반드시 **Template 기반**으로 변환해야 한다.
> Raw 로그 그대로는 패턴 탐지 불가능.

---

### ② Feature Layer (여기서 승부 남)

3가지를 **동시에** 생성한다.

| Feature | 예시 | 용도 |
|---|---|---|
| **Sequence** | `[A, B, C, ERROR]` | 이벤트 흐름 분석 |
| **Frequency** | `A: 1200회 / ERROR: 3회` | 이상 빈도 탐지 |
| **Embedding** | 벡터 변환 | 의미 기반 유사 검색 |

---

### ③ 저장 구조 (분리 필수)

```mermaid
flowchart LR
    VDB["Vector DB\n의미 검색"]
    PDB["Pattern DB\n패턴 / 빈도"]
    RAW["Raw Storage\n원본 복구"]

    style VDB fill:#1e3a5f,stroke:#60a5fa,color:#e2e8f0
    style PDB fill:#1a3b1a,stroke:#4ade80,color:#e2e8f0
    style RAW fill:#3b2a00,stroke:#fbbf24,color:#fde68a
```

> **이 3개를 하나로 합치면 성능이 망가진다.**

---

## 3. Query 처리 구조

**입력 예시**
```
"장애 발생 전 공통 패턴 찾아줘"
```

```mermaid
flowchart TD
    Q([사용자 질의]) --> PLAN["① LLM Planner (선택)\n→ ERROR 기준\n→ 이전 5분 window\n→ 특정 device"]

    PLAN --> HS["② Hybrid Search"]
    HS --> V["Vector Search\n유사 로그"]
    HS --> F["Filter\ndevice_id, ERROR"]
    HS --> P["Pattern 조회\n빈도 높은 시퀀스"]

    V --> SEQ_R["③ 시퀀스 재구성\ntimestamp 정렬\n→ 이벤트 흐름 생성"]
    F --> SEQ_R
    P --> SEQ_R

    SEQ_R --> ANAL["④ 패턴 분석\nSequential Mining\nFrequency 분석\n(선택) LLM 해석"]

    ANAL --> OUT([결과 출력])
```

---

## 4. 패턴 탐지 엔진

### 구조

```
Sequence 데이터
 → PrefixSpan (시퀀스 패턴)
 → 빈도 분석
 → 이상 패턴 탐지
```

### 결과 예시

```
정상 패턴:
  A → B → C

이상 패턴:
  A → B → X → ERROR
```

---

## 5. 3단계 구현 전략

```mermaid
flowchart LR
    subgraph S1["1단계 · MVP (필수)"]
        D1[Template Parsing\nDrain]
        D2[Sequence 생성]
        D3[Frequency 분석]
    end

    subgraph S2["2단계 · 패턴 분석"]
        D4[PrefixSpan]
        D5[Clustering]
    end

    subgraph S3["3단계 · AI 추가"]
        D6[Vector DB]
        D7[LLM Planner\n+ Analyzer]
    end

    S1 --> S2 --> S3
```

| 단계 | 내용 | 효과 |
|---|---|---|
| **1단계 MVP** | Drain + Sequence + Frequency | 전체 문제의 **70% 해결** |
| **2단계** | PrefixSpan + Clustering | 고급 패턴 탐지 |
| **3단계** | Vector DB + LLM | AI 기반 자동 분석 |

---

## 6. 기술 스택

| 영역 | 기술 |
|---|---|
| **로그 파싱** | Drain3 (Python) |
| **패턴 분석** | prefixspan, pandas |
| **Vector DB** | FAISS / Qdrant |
| **저장소** | PostgreSQL / ClickHouse |

---

## 7. 핵심 설계 철학

> **1. 로그 = 텍스트가 아니라 이벤트다**
> **2. 패턴 = 단어가 아니라 시퀀스다**
> **3. 분석 = 검색이 아니라 구조화다**

---

## 8. 최종 압축 플로우

```mermaid
flowchart LR
    LOG([Log]) --> T[Template]
    T --> S[Sequence]
    T --> F[Frequency]
    T --> E[Embedding]
    S & F & E --> STORE[(저장)]

    Q([Query]) --> LP[LLM Planner]
    LP --> HS[Hybrid Search]
    STORE --> HS
    HS --> SA[Sequence 분석]
    SA --> OUT([패턴 도출])
```

---

## 9. 한 줄 결론

> **"Template 기반으로 로그를 이벤트화하고, 시퀀스/빈도/벡터를 결합해 패턴을 찾는 하이브리드 구조가 최적"**

---

## 다음 단계

```
Python 최소 구현 (바로 실행 가능)
  → Drain3  (Template Parsing)
  → PrefixSpan  (Sequence Pattern Mining)
  → FAISS  (Vector Search)
```
