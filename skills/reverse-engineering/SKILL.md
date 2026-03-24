---
name: reverse-engineering
description: Use when understanding an existing codebase before modifying it, or documenting undocumented code structure and flows. Trigger on "코드베이스 분석", "코드 구조 파악", "이 코드 어떻게 동작해", "프로젝트 분석", "기존 코드 이해", "코드 읽어줘", "analyze codebase", "reverse engineer", "understand this code", "흐름 추적", "기술 부채 파악", "모듈 구조". Not for writing new code or debugging — use brainstorming or systematic-debugging instead.
metadata:
  version: 0.3.0
  author: Jay
  category: analysis
  invoke_mode: user-invocable
  return_behavior: stop-no-gate
---

# reverse-engineering

<!-- 기존 코드베이스의 구조와 동작을 분석하여 문서화하는 스킬 -->
<!-- 구조 패턴: Pipeline (주) + Generator (산출물 템플릿) -->
<!-- 행동 패턴: User-Invocable (주) + Three-Mode (Lite/Deep/Delta) -->
<!-- 자유도: Phase 1~2 저자유도 (탐색 전략 고정) / Phase 3~4 중자유도 (분석 관점 재량) -->

## Trigger

다음 상황에서 이 스킬을 실행한다:

- 기존 코드베이스를 처음 접하여 전체 구조를 파악해야 할 때
- 특정 모듈/서브시스템의 동작 원리를 이해해야 할 때
- 코드 변경 전 영향 범위를 파악해야 할 때
- 문서화되지 않은 코드를 문서화해야 할 때
- aidlc INCEPTION에서 Brownfield 감지 후 분석이 제안되었을 때

## Purpose

기존 코드베이스의 구조, 흐름, 패턴을 분석하여 문서화한다. 모든 결론에 근거(파일:라인)와 confidence 태그를 부착하고, 미탐색 범위를 명시적으로 선언한다.

---

## 입력

| 항목 | 필수 | 설명 |
|------|------|------|
| 대상 경로 | 선택 | 분석할 디렉토리/모듈. 미지정 시 프로젝트 루트 |
| 모드 | 선택 | Lite / Deep / Delta. 기본 Lite |
| 관심 영역 | 선택 | 특정 흐름, 모듈, 기능 (Deep 모드에서 유용) |

---

## 탐색 원칙

모든 Phase에서 적용하는 토큰 효율 전략:

1. **매니페스트 우선**: 코드 읽기 전 매니페스트/설정 파일로 전체상 파악
2. **Guided DFS**: grep 심볼 -> 호출 추적 -> 외부 경계에서 중지. 무작위 탐색 금지
3. **Sampling**: 대규모 디렉토리에서 대표 파일 1~2개만 상세 분석
4. **예산 기반 점진 심화**: confidence가 low인 결론만 추가 탐색
5. **Delta 모드**: 기존 산출물 재활용, 변경분만 분석

---

## 프로세스

### Step 0: 모드 선택

기존 산출물(`reverse-engineering/`)이 존재하면 Delta를 제안. 미지정 시 기본 Lite.

```
분석 모드:
L) Lite -- 전체 구조 요약 (진입점 + 주요 흐름 3개)
D) Deep -- 심층 분석 (모든 Phase + evidence 포함)
d) Delta -- 기존 산출물 기반 증분 분석
```

**Delta 모드 diff 범위**: 기존 산출물의 "분석 일시"와 현재 HEAD 사이의 diff를 기본으로 사용한다. 분석 일시가 없으면 사용자에게 범위를 확인한다.

**모드별 실행 범위:**

| Phase | Lite | Deep | Delta |
|-------|:----:|:----:|:-----:|
| Phase 1: 구조 추출 | 전체 | 전체 | 변경분만 |
| Phase 2: 흐름 추적 | 진입점 3개 | 전체 | 변경분만 |
| Phase 3: 패턴/부채 | 스킵 | 전체 | 변경분만 |
| Phase 4: 검증 | 스킵 | 전체 | 변경분만 |
| 산출물 | README + diagram | README + diagram + evidence | 기존 산출물 업데이트 |

> **DO NOT proceed to Phase 2 until Phase 1 is complete.** 구조를 모르고 흐름을 추적하면 누락이 발생한다.

---

### Phase 1: 구조 추출 (Structure Extraction)

> 자유도: **저**. 아래 순서를 반드시 따른다. 순서를 바꾸면 진입점을 놓친다.

#### 1.1 매니페스트 분석 (코드 읽기 전 반드시 먼저)

매니페스트 파일에서 기술 스택과 의존성을 추출한다.

- `package.json`, `pyproject.toml`, `build.gradle.kts`, `Cargo.toml`, `go.mod` 등
- 주요 의존성 분류: 프레임워크, DB, 외부 API, 테스트

#### 1.2 진입점 식별

코드 실행의 시작점들을 찾는다.

- main 함수 / 앱 부트스트랩
- API routes / controllers
- Event handlers / message consumers
- CLI commands
- Scheduled jobs / cron

**함정 주의 — 동적 import/라우트**: `require(variable)`, `importlib.import_module()`, 파일 기반 라우팅(Next.js `app/`, Nuxt `pages/`) 등은 grep으로 잡히지 않는다. 매니페스트에서 프레임워크를 먼저 확인하고, 해당 프레임워크의 라우팅 컨벤션을 적용하여 진입점을 보완한다.

#### 1.3 모듈 의존성 그래프

디렉토리/패키지 간 의존 관계를 추적한다.

- import/require/use 문 분석
- 순환 의존성 표시
- 외부 라이브러리 경계 표시

**함정 주의 — 순환 의존성 추적 시 무한 루프**: A -> B -> C -> A 형태 발견 시 순환 경로를 기록하고 더 이상 추적하지 않는다. 순환 내부를 반복 탐색하지 않는다.

#### 1.4 데이터 모델/스키마 추출

- DB 스키마 (마이그레이션 파일, ORM 모델)
- 전역 상태 관리 (Redux store, Context, 싱글톤)
- 주요 도메인 엔티티

---

### Phase 2: 흐름 추적 (Flow Tracing)

> 자유도: **저**. Guided DFS 전략을 따른다. 무작위 탐색 금지.

> **DO NOT proceed to Phase 3 until Phase 2 is complete.** 흐름을 모르면 패턴 식별이 표면적이 된다.

#### 2.1 주요 흐름 선택

Phase 1의 진입점에서 출발하여 가장 중요한 흐름을 선택한다.

- Lite: 최대 3개
- Deep: 관심 영역 중심 + 주요 흐름 전체

#### 2.2 Guided Depth-First Search

각 흐름에 대해:

1. grep으로 심볼 정의/사용처 수집
2. 호출 체인 추적 (caller -> callee)
3. **외부 라이브러리 경계에서 중지** — 라이브러리 내부로 진입하지 않는다
4. 데이터 변환 과정 기록 (input -> processing -> output)

#### 2.3 Cross-cutting Concerns 식별

흐름 추적 중 발견되는 공통 관심사를 README의 "Cross-cutting Concerns" 섹션에 기록한다. Deep 모드에서는 각 항목의 근거(파일:라인)도 evidence.md에 함께 기록한다.

대상: 에러 핸들링 패턴, 로깅/모니터링, 인증/인가, 트랜잭션 관리, 캐싱

---

### Phase 3: 패턴/부채 식별 (Pattern & Debt Identification)

> Lite 모드에서는 스킵.
> 자유도: **중**. 아래 관점을 참조하되, 코드베이스 특성에 따라 관점을 추가/생략할 수 있다.

#### 3.1 코딩 패턴/컨벤션

네이밍 규칙, 에러 처리 방식, 디자인 패턴, 테스트 작성 패턴

#### 3.2 기술 부채

복잡도 높은 구간, 안티패턴, 중복 코드, TODO/FIXME/HACK 주석

#### 3.3 암묵적 아키텍처 결정 (Implicit ADRs)

코드에서 읽히지만 문서화되지 않은 설계 결정. git blame/log에서 단서 수집.

#### 3.4 테스트 커버리지 현황

테스트 파일 구조, 테스트 대상 vs 미대상 모듈, 테스트 종류 (단위/통합/E2E)

---

### Phase 4: 검증 (Verification)

> Lite 모드에서는 스킵.
> 자유도: **중**. 검증 방법은 코드베이스에 맞게 조정 가능하나, confidence 태그와 미탐색 선언은 필수.

#### 4.1 검증 쿼리

분석 결과의 완전성을 자체 점검:

- "누락된 엔드포인트/이벤트가 있는가?"
- "식별되지 않은 진입점이 있는가?"
- "데이터 흐름에 빠진 경로가 있는가?"

#### 4.2 Hypothesis Testing

불확실한 결론에 대해 가설 -> 코드 확인 -> 결과(confirmed/refuted/inconclusive)를 evidence.md에 기록.

#### 4.3 Confidence 태그 부착 (필수)

모든 결론에 신뢰도를 명시한다:

| 태그 | 기준 |
|------|------|
| `high` | 코드에서 직접 확인됨 (파일:라인 근거) |
| `medium` | 패턴/컨벤션에서 추론, 직접 확인은 부분적 |
| `low` | 추정. 관련 코드 미탐색 또는 근거 불충분 |

#### 4.4 미탐색 범위 선언 (필수)

분석하지 못한 영역을 evidence.md의 "미탐색 범위" 테이블에 기록한다. 이유: 접근 불가, 시간 제약, 도메인 지식 필요.

---

### Step 5: 산출물 생성

이 스킬 디렉토리의 `assets/output-template.md`를 로드하여 README.md를 채운다. Deep 모드에서는 `assets/evidence-template.md`를 로드하여 evidence.md도 생성한다.

Mermaid diagram 유형 선택:
- **모듈 관계**: `flowchart` (의존성 방향 표현)
- **데이터 흐름/API 호출 순서**: `sequenceDiagram` (시간순 상호작용)
- **상태 머신**: `stateDiagram-v2` (상태 전이)
- **여러 유형이 필요하면** diagram.mermaid에 복수 다이어그램을 포함한다

#### 저장 경로

```
reverse-engineering/
├── README.md       -- 요약, Entry Point Map, 리스크, open questions
├── diagram.mermaid -- 모듈 관계 + 주요 흐름 시각화
└── evidence.md     -- 근거, confidence 태그, 미탐색 범위 (Deep만)
```

---

## aidlc-devflow 연동

이 스킬은 독립적으로 동작한다. aidlc-devflow가 설치된 환경에서의 연동:

| 시점 | 동작 |
|------|------|
| INCEPTION workspace-detection에서 Brownfield 감지 | "reverse-engineering 분석할까요?" 제안 |
| 이 플러그인 미설치 | 제안 없이 기존 흐름 진행 (graceful degradation) |
| 분석 완료 | 산출물을 requirements-analysis 입력으로 전달 |

**트리거 경계**: workspace-detection은 "이 프로젝트의 기술 스택은 무엇인가"(What)를 판별한다. reverse-engineering은 "이 코드는 어떻게 동작하는가"(How)를 분석한다. 두 스킬은 상호 배타가 아니라 순차적이다 — workspace-detection 완료 후 reverse-engineering이 실행된다.

---

## Examples

### Example 1: Lite 모드 -- 새 프로젝트 합류

```
사용자: "이 코드베이스 분석해줘"

[모드: Lite, 대상: 프로젝트 루트]
-> Phase 1: 구조 추출
  - 1.1 package.json 분석 -> Next.js 14 (app router) + Prisma + PostgreSQL
  - 1.2 진입점 식별: API routes 3개 + app/ 하위 pages 2개
    - 판단: Next.js app router이므로 파일 기반 라우팅 -> app/ 디렉토리 스캔
  - 1.3 모듈 6개, 순환 의존 0
  - 1.4 Prisma schema에서 도메인 엔티티 4개 추출
-> Phase 2: 주요 흐름 3개 추적 (Lite 상한)
  - POST /api/auth/login: NextAuth -> Prisma -> bcrypt 검증 -> JWT 발급
  - GET /api/users: middleware(auth) -> Prisma.findMany -> pagination
  - POST /api/orders: auth -> 입력 검증 -> Prisma.create -> 이메일 알림
  - Cross-cutting: NextAuth 미들웨어가 /api/auth 제외 모든 라우트에 적용
-> 산출물: README.md + diagram.mermaid (flowchart: 모듈 의존성)
```

### Example 2: Deep 모드 -- 특정 모듈 심층 분석

```
사용자: "src/payment 모듈을 깊이 분석해줘"

[모드: Deep, 대상: src/payment/, 관심 영역: 결제 흐름]
-> Phase 1: 구조 추출
  - 진입점: PaymentController.processPayment(), webhook handler
  - 외부 의존: stripe SDK, 내부 의존: UserService, OrderService
-> Phase 2: 흐름 추적
  - 결제 흐름: Controller -> PaymentService.charge() -> Stripe API -> webhook callback
  - 판단: webhook handler에서 idempotency key 미사용 발견 -> Phase 3 기술 부채 기록 예정
-> Phase 3: 패턴/부채
  - 패턴: Repository 패턴 (PaymentRepository), Strategy 패턴 (PG사 어댑터)
  - 부채: webhook idempotency 미구현 (중복 결제 위험), 에러 재시도 로직 없음
  - Implicit ADR: git blame -> PG사를 Stripe에서 Toss로 전환 시도 흔적 (6개월 전 revert)
-> Phase 4: 검증
  - 가설: "모든 결제 경로가 PaymentService를 거친다"
    -> grep PaymentService 사용처 -> refund 경로가 직접 Stripe API 호출 (가설 refuted)
  - confidence 분포: high 15, medium 6, low 2
  - 미탐색: Stripe Dashboard 설정 (외부 시스템, 접근 불가)
-> 산출물: README + diagram (sequenceDiagram: 결제 흐름) + evidence (23개 근거)
```

### Example 3: Delta 모드 -- 변경 후 업데이트

```
사용자: "지난주 변경사항 반영해서 분석 업데이트해줘"

[모드: Delta, 기존 산출물 존재, 분석 일시: 2026-03-17]
-> git diff 2026-03-17..HEAD --stat: 변경 파일 8개 (src/auth/ 3, src/api/ 5)
-> Phase 1 (변경분): auth 모듈에 OAuth provider 1개 추가 -> 진입점 1개 추가
-> Phase 2 (변경분): OAuth 콜백 흐름 새로 추적
-> 기존 산출물 업데이트: README Entry Point Map에 1행 추가, diagram에 OAuth 노드 추가
```

---

## Troubleshooting

### 동적 import/파일 기반 라우팅에서 진입점 누락

**증상**: Phase 1.2에서 grep 기반 탐색이 진입점을 찾지 못함
**원인**: Next.js app router, Nuxt pages/, `importlib.import_module()` 등 동적 패턴
**해결**: 1.1에서 식별한 프레임워크의 라우팅 컨벤션을 적용. 예: Next.js면 `app/**/page.tsx`를 Glob으로 스캔

### 순환 의존성에서 추적이 끝나지 않음

**증상**: Phase 1.3 모듈 의존성 추적이 같은 모듈을 반복 방문
**원인**: A -> B -> C -> A 순환 의존
**해결**: 이미 방문한 모듈을 기록하고, 재방문 시 "순환 의존: A -> B -> C -> A" 형태로 기록 후 중단. 순환 내부를 반복 탐색하지 않는다

### monorepo에서 진입점이 과다 식별됨

**증상**: Phase 1.2에서 수십 개의 진입점이 나옴
**원인**: monorepo 내 여러 패키지/서비스가 각각 진입점을 가짐
**해결**: 대상 경로를 특정 패키지로 좁힌다. 예: "이 코드베이스 분석해줘" 대신 "packages/api 분석해줘"

### Delta 모드에서 기존 산출물을 찾지 못함

**증상**: "기존 산출물이 없습니다" 메시지
**해결**: `reverse-engineering/` 디렉토리 경로 확인. 다른 위치에 있으면 경로 지정

### confidence가 low인 결론이 많음

**증상**: evidence.md에 low 태그가 과반
**해결**: 해당 영역을 Deep 모드로 재분석하거나, 도메인 전문가에게 open questions 확인
