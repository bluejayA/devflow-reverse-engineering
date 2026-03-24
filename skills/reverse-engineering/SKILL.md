---
name: reverse-engineering
description: Use when analyzing an existing codebase to understand its structure, flows, patterns, and technical debt. Trigger on "코드베이스 분석", "코드 구조 파악", "reverse engineer", "analyze codebase", "흐름 추적", "기존 코드 이해".
metadata:
  version: 0.1.0
  author: Jay
  category: analysis
  invoke_mode: user-invocable
  return_behavior: stop-no-gate
---

# reverse-engineering

<!-- Brownfield 코드베이스의 구조, 흐름, 패턴을 분석하여 문서화 -->

## Trigger

다음 상황에서 이 스킬을 실행한다:

- 기존 코드베이스를 처음 접할 때
- 특정 모듈/서브시스템의 동작을 이해해야 할 때
- 코드 변경 전 영향 범위를 파악해야 할 때
- 기술 부채를 식별하고 문서화해야 할 때
- aidlc INCEPTION에서 Brownfield 감지 후 분석이 제안되었을 때

## Purpose

Brownfield 코드베이스의 구조, 흐름, 패턴을 분석하여 3층 산출물(README + diagram + evidence)로 문서화한다. 모든 결론에 근거(파일:라인)와 confidence 태그를 부착하고, 미탐색 범위를 명시적으로 선언한다.

---

## 입력

| 항목 | 필수 | 설명 |
|------|------|------|
| 대상 경로 | 선택 | 분석할 디렉토리/모듈. 미지정 시 프로젝트 루트 |
| 모드 | 선택 | Lite / Deep / Delta. 기본 Lite |
| 관심 영역 | 선택 | 특정 흐름, 모듈, 기능 (Deep 모드에서 유용) |

---

## 실행 모드

### Lite 모드

전체 구조 요약. INCEPTION에서 Brownfield + Standard/Comprehensive 시 오케스트레이터가 제안.

- Phase 1(구조 추출)만 전체 실행
- Phase 2(흐름 추적)는 주요 진입점 3개까지만
- Phase 3~4는 스킵
- 산출물: README.md + diagram.mermaid (evidence 생략)

### Deep 모드

사용자가 직접 호출. 특정 서브시스템/흐름 지정 가능.

- Phase 1~4 전체 실행
- 관심 영역 지정 시 해당 영역 중심으로 깊이 분석
- 산출물: 3층 전체 (README + diagram + evidence)

### Delta 모드

기존 산출물 존재 시 git diff 기반 증분 분석.

- 기존 산출물(`reverse-engineering/`)을 읽어 현재 상태 파악
- `git diff` 또는 `git log --since` 기반으로 변경된 파일/모듈 식별
- 변경 영역만 Phase 1~4 재실행
- 기존 산출물을 업데이트 (전체 재생성 아님)

---

## 프로세스

### 0. 모드 확인

모드가 지정되지 않았으면 기본 Lite. 기존 산출물(`reverse-engineering/`)이 존재하면 Delta를 제안.

```
분석 모드:
L) Lite -- 전체 구조 요약
D) Deep -- 심층 분석 (관심 영역 지정 가능)
d) Delta -- 기존 산출물 기반 증분 분석
```

---

### Phase 1: 구조 추출 (Structure Extraction)

프로젝트의 뼈대를 파악한다.

#### 1.1 매니페스트 분석

프로젝트 매니페스트 파일에서 기술 스택과 의존성을 추출한다.

- `package.json`, `pyproject.toml`, `build.gradle.kts`, `Cargo.toml`, `go.mod` 등
- 주요 의존성 분류: 프레임워크, DB, 외부 API, 테스트

#### 1.2 진입점 식별

코드 실행의 시작점들을 찾는다.

- main 함수 / 앱 부트스트랩
- API routes / controllers
- Event handlers / message consumers
- CLI commands
- Scheduled jobs / cron

#### 1.3 모듈 의존성 그래프

디렉토리/패키지 간 의존 관계를 추적한다.

- import/require/use 문 분석
- 순환 의존성 표시
- 외부 라이브러리 경계 표시

#### 1.4 데이터 모델/스키마 추출

- DB 스키마 (마이그레이션 파일, ORM 모델)
- 전역 상태 관리 (Redux store, Context, 싱글톤)
- 주요 도메인 엔티티

---

### Phase 2: 흐름 추적 (Flow Tracing)

주요 유즈케이스의 엔드투엔드 흐름을 추적한다.

#### 2.1 주요 흐름 선택

진입점에서 출발하여 가장 중요한 흐름 N개를 선택한다.

- Lite: 최대 3개
- Deep: 관심 영역 중심 + 주요 흐름 전체

#### 2.2 Guided Depth-First Search

각 흐름에 대해:

1. grep으로 심볼 정의/사용처 수집
2. 호출 체인 추적 (caller → callee)
3. 외부 라이브러리 경계에서 중지
4. 데이터 변환 과정 기록 (input → processing → output)

#### 2.3 Cross-cutting Concerns 식별

흐름 추적 중 발견되는 공통 관심사:

- 에러 핸들링 패턴
- 로깅/모니터링
- 인증/인가
- 트랜잭션 관리
- 캐싱

---

### Phase 3: 패턴/부채 식별 (Pattern & Debt Identification)

> Lite 모드에서는 스킵.

#### 3.1 코딩 패턴/컨벤션 추출

- 네이밍 규칙
- 에러 처리 방식
- 디자인 패턴 사용 (Repository, Factory, Observer 등)
- 테스트 작성 패턴

#### 3.2 기술 부채 지점

- 복잡도 높은 구간 (긴 함수, 깊은 중첩)
- 안티패턴 (God class, 순환 의존, magic numbers)
- 중복 코드
- TODO/FIXME/HACK 주석

#### 3.3 암묵적 아키텍처 결정 (Implicit ADRs)

코드에서 읽히지만 문서화되지 않은 설계 결정:

- "왜 이 라이브러리를 선택했는가"
- "왜 이 구조인가"
- git blame/log에서 단서 수집

#### 3.4 테스트 커버리지 현황

- 테스트 파일 존재 여부 / 구조
- 테스트 대상 vs 미대상 모듈
- 테스트 종류 (단위/통합/E2E)

---

### Phase 4: 검증 (Verification)

> Lite 모드에서는 스킵.

#### 4.1 검증 쿼리

분석 결과의 완전성을 자체 점검한다:

- "누락된 엔드포인트/이벤트가 있는가?"
- "식별되지 않은 진입점이 있는가?"
- "데이터 흐름에 빠진 경로가 있는가?"

#### 4.2 Hypothesis Testing

불확실한 결론에 대해 가설을 세우고 코드에서 확인한다:

```
가설: AuthMiddleware가 모든 API 라우트에 적용된다
검증: app.use(authMiddleware) 위치 확인 → router 등록 순서 확인
결과: [confirmed/refuted/inconclusive]
```

#### 4.3 Confidence 태그 부착

모든 결론에 신뢰도를 명시한다:

| 태그 | 기준 |
|------|------|
| `high` | 코드에서 직접 확인됨 (파일:라인 근거) |
| `medium` | 패턴/컨벤션에서 추론, 직접 확인은 부분적 |
| `low` | 추정. 관련 코드 미탐색 또는 근거 불충분 |

#### 4.4 미탐색 범위 선언

분석하지 못한 영역을 명시적으로 선언한다:

- 접근 불가: 외부 서비스, 비공개 패키지
- 시간 제약: 대규모 모듈 중 샘플링만 수행
- 이해 부족: 도메인 지식 필요

---

## 산출물

### 저장 경로

```
reverse-engineering/
├── README.md       -- 요약, Entry Point Map, 리스크, open questions
├── diagram.mermaid -- 모듈 관계 + 주요 흐름 시각화
└── evidence.md     -- 근거 파일:라인, confidence 태그, 미탐색 범위
```

### README.md 구조

```markdown
# Reverse Engineering: [프로젝트명]

## 분석 메타데이터
- 분석 일시: [timestamp]
- 모드: [Lite|Deep|Delta]
- 대상: [경로]

## 기술 스택 요약
[매니페스트 기반 스택 정리]

## Entry Point Map
| 진입점 | 유형 | 위치 | 설명 |
|--------|------|------|------|

## 모듈 구조
[주요 모듈/패키지 설명 + 의존 관계 요약]

## 주요 흐름
### 흐름 1: [이름]
[단계별 설명]

## Cross-cutting Concerns
[공통 관심사 정리]

## 패턴/컨벤션 (Deep만)
[발견된 패턴 정리]

## 기술 부채 / 리스크 (Deep만)
[부채 지점 + 심각도]

## Open Questions
[미해결 질문 목록]
```

### diagram.mermaid

모듈 관계 + 주요 흐름을 Mermaid로 시각화한다.

### evidence.md (Deep만)

```markdown
# Evidence Log

## 결론별 근거

### [결론 제목] [confidence: high|medium|low]
- 근거: [파일:라인] -- [설명]
- 근거: [파일:라인] -- [설명]

## 미탐색 범위
| 영역 | 이유 | 영향 |
|------|------|------|
```

---

## 토큰 효율 전략

1. **매니페스트 우선**: 코드 읽기 전 매니페스트/설정 파일로 전체상 파악
2. **Guided DFS**: grep 심볼 → 호출 추적 → 외부 경계에서 중지. 무작위 탐색 금지
3. **Sampling**: 대규모 디렉토리에서 대표 파일 1~2개만 상세 분석
4. **예산 기반 점진 심화**: confidence가 low인 결론만 추가 탐색
5. **Delta 모드**: 기존 산출물 재활용, 변경분만 분석

---

## aidlc-devflow 연동

이 스킬은 독립적으로 동작한다. aidlc-devflow가 설치된 환경에서의 연동:

| 시점 | 동작 |
|------|------|
| INCEPTION workspace-detection에서 Brownfield 감지 | "reverse-engineering 분석할까요?" 제안 |
| 이 플러그인 미설치 | 제안 없이 기존 흐름 진행 |
| 분석 완료 | 산출물을 requirements-analysis 입력으로 전달 |

---

## Examples

### Example 1: Lite 모드 -- 새 프로젝트 합류

```
사용자: "이 코드베이스 분석해줘"

[모드: Lite, 대상: 프로젝트 루트]
-> Phase 1: 구조 추출
  - package.json 분석 -> Next.js + Prisma + PostgreSQL
  - 진입점 5개 식별 (API routes 3, pages 2)
  - 모듈 6개, 순환 의존 0
-> Phase 2: 주요 흐름 3개 추적
  - POST /api/auth/login -> 인증 흐름
  - GET /api/users -> 사용자 목록 흐름
  - POST /api/orders -> 주문 생성 흐름
-> 산출물: README.md + diagram.mermaid
```

### Example 2: Deep 모드 -- 특정 모듈 심층 분석

```
사용자: "src/payment 모듈을 깊이 분석해줘"

[모드: Deep, 대상: src/payment/, 관심 영역: 결제 흐름]
-> Phase 1~4 전체 실행
-> 결과: 3층 산출물
  - README: 결제 모듈 구조 + 4개 흐름 + PG사 연동 패턴
  - diagram: 결제 상태 머신 + 외부 API 호출 순서
  - evidence: 23개 근거 (high 15, medium 6, low 2), 미탐색 1건
```

### Example 3: Delta 모드 -- 변경 후 업데이트

```
사용자: "지난주 변경사항 반영해서 분석 업데이트해줘"

[모드: Delta, 기존 산출물 존재]
-> git diff --stat HEAD~15: 변경 파일 8개 (src/auth/ 3, src/api/ 5)
-> Phase 1~4를 변경 영역(auth, api)에 대해서만 실행
-> 기존 산출물 업데이트: README 2개 섹션 수정, diagram 1개 노드 추가
```

---

## Troubleshooting

### 대규모 코드베이스에서 분석이 느림

**증상**: 파일이 수백 개 이상이라 Phase 1에서 시간 소모
**해결**: Lite 모드로 시작 + 관심 영역을 좁혀서 Deep 모드 재실행

### Delta 모드에서 기존 산출물을 찾지 못함

**증상**: "기존 산출물이 없습니다" 메시지
**해결**: `reverse-engineering/` 디렉토리 경로 확인. 다른 위치에 있으면 경로 지정

### confidence가 low인 결론이 많음

**증상**: evidence.md에 low 태그가 과반
**해결**: 해당 영역을 Deep 모드로 재분석하거나, 도메인 전문가에게 open questions 확인
