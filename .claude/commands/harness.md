이 프로젝트는 Harness 프레임워크를 사용한다. 아래 워크플로우에 따라 작업을 진행하라.

---

## 워크플로우

### 1. 다음 기능 확인

`TODO.md`를 읽고 첫 번째 `- [ ]` 최상위 항목과 그 하위 항목 전체를 추출한다.

- 모든 항목이 `[x]`이면 "모든 기능이 완료되었습니다." 출력 후 종료.
- `phases/index.json`을 읽어 해당 기능의 slug가 없으면 pending으로 신규 등록한다.
  - slug: TODO 항목 제목을 영문 kebab-case로 변환 (예: "로그인 기능 구현" → "login-feature")
  - index.json에 `{ "slug": "...", "name": "...", "status": "pending" }` 추가

### 2. Plan 에이전트

Agent 툴로 실행하라:
- **name**: `"plan-agent"`
- **description**: `"Plan 에이전트 — {기능명} plan.md 작성"`
- **model**: `opus`

다음 파일들을 읽고 구현 계획을 수립하는 에이전트를 실행하라:

**읽어야 할 파일:**
- `CLAUDE.md`
- `docs/PROJECT.md`
- `docs/LESSONS.md`
- `phases/index.json` — 완료된 기능의 summary 목록 확인
  - summary를 보고 현재 기능과 관련 있는 기능의 `phases/{slug}/context.md`만 선택해서 읽기

**에러 재시도인 경우:**
- index.json의 해당 기능에 `error_summary`가 있으면 원인을 분석한다.
- API 키, 환경변수, 외부 설정 등 사용자 개입이 필요한 원인이라면 먼저 사용자에게 해결 방법을 질문하고 답변을 반영해서 plan.md를 작성한다.

**plan.md 작성 규칙:**
- 기능 범위는 TODO 항목에 명시된 것만 다룬다. 항목 추가는 가능하나 기존 항목 수정/삭제는 금지.
- 계획 수립 중 TODO에 없는 세부 항목이 필요하다고 판단되면 TODO.md 해당 최상위 항목 하위에 추가하고 index.json에도 반영한 뒤 "추가된 항목: ..." 형태로 사용자에게 명시한다.

**plan.md 형식 (`phases/{slug}/plan.md`):**
```markdown
# {기능명} 구현 계획

## 구현 범위
TODO 항목 기반으로 이번에 구현할 것 목록

## 구현할 파일
- `src/main/java/.../controller/FooController.java` — 역할 설명
- `src/main/java/.../service/FooService.java`
- `src/main/java/.../repository/FooRepository.java`
- `src/test/java/.../FooServiceTest.java`

## 핵심 인터페이스
클래스/메서드 시그니처 수준만 제시 (구현 없음)

## API 엔드포인트 (해당 시)
- `POST /api/v1/foo` — 요청/응답 구조

## 구현 순서
1. 테스트 작성 (Service 메서드 직접 호출하는 통합 테스트)
2. Repository → Service → Controller 순으로 구현
3. 테스트 통과 확인

## 의존하는 기존 모듈
이전 기능의 context.md에서 파악한 관련 파일들 (예: JwtAuthenticationFilter 등)

## 주의사항
- X를 하지 마라. 이유: Y
- 모든 예외는 ApplicationException 상속
- 컨트롤러 반환 타입은 ResponseDTO<T> 사용
```

### 3. 사용자 plan.md 검토

plan.md 작성 완료 후 사용자에게 검토를 요청한다.

- 승인하면 4번으로 진행.
- 수정 요청이 있으면 plan.md를 수정하고 다시 검토를 요청한다.

### 4. 테스트 에이전트

Agent 툴로 실행하라:
- **name**: `"test-agent"`
- **description**: `"테스트 에이전트 — {기능명} 실패 테스트 작성"`
- **model**: `opus`

`phases/{slug}/plan.md`를 읽고 테스트 파일만 작성하는 에이전트를 실행하라.

**규칙:**
- 테스트 파일만 작성한다. 구현 코드 작성 금지.
- Service 메서드를 직접 호출하는 통합 테스트로 작성한다.
- plan.md의 구현 범위에 있는 모든 케이스가 테스트로 존재해야 한다. 정상 케이스, 에러 케이스, 엣지 케이스, 예외 처리 포함.
- 테스트 케이스에 주석으로 "어떤 동작을 검증하는 테스트인지" 명시한다.
- 수동으로 확인할 수 있는 케이스는 주석에 샘플 요청/응답 데이터 포함.

테스트 파일이 하나도 없거나 plan.md 요구사항 대비 누락된 케이스가 있으면 이후 단계 진행 불가. 즉시 중단하고 사용자에게 보고한다.

### 5. 구현 에이전트

Agent 툴로 실행하라:
- **name**: `"impl-agent"`
- **description**: `"구현 에이전트 — {기능명} 테스트 통과 구현"`
- **model**: `opus`

`phases/{slug}/plan.md`와 4번에서 작성된 테스트 파일을 읽고 구현하는 에이전트를 실행하라.

**재시도인 경우 (6번 리뷰에서 돌아온 경우):**
- `phases/{slug}/review-feedback.md`를 읽고 지적 사항을 확인한 뒤 수정한다.

**규칙:**
- 테스트를 통과하는 구현 코드만 작성한다.
- **테스트 파일 수정 절대 금지.** 테스트가 실패한다면 테스트를 바꾸는 게 아니라 구현을 바꿔야 한다. 테스트 파일을 수정하는 것은 요구사항을 임의로 변경하는 것과 같다.
- 구현 완료 후 테스트를 실행해서 통과 여부를 확인한다.

### 6. 리뷰 에이전트

Agent 툴로 실행하라:
- **name**: `"review-agent"`
- **description**: `"리뷰 에이전트 — {기능명} 코드 품질 검토"`
- **model**: `opus`

구현된 코드 전체와 테스트 파일을 검토하는 에이전트를 실행하라.

**검토 기준 (10년차 시니어 개발자 관점):**
- 클린 아키텍처 준수 여부
- 확장성과 유지보수성
- 코드 품질 (중복, 불필요한 복잡도)
- 테스트가 plan.md 요구사항 기반으로 작성됐는지 (구현에 맞게 끼워 맞춘 테스트가 아닌지)

**검토 완료 후 항상 `phases/{slug}/review-feedback.md`를 작성한다** (통과/실패 무관):
```markdown
# 리뷰 결과 — {기능명} (N회차)

## 결과
통과 / 실패

## 검토 항목
- 항목: 통과 여부 및 코멘트

## 지적 사항 (실패 시)
- 파일명:라인: 구체적인 문제와 수정 방향
```

**결과:**
- 문제 없음 → 7번으로 진행.
- 문제 있음 → 5번(구현 에이전트)으로 돌아가 수정. **최대 3회.**
- 3회 후에도 실패 → `phases/index.json`의 해당 기능에 `"status": "error"`, `"error_summary": "한 줄 요약"` 기록 후 중단. 사용자에게 에러 내용 출력.

### 7. Lint 에이전트

Agent 툴로 실행하라:
- **name**: `"lint-agent"`
- **description**: `"Lint 에이전트 — {기능명} 포맷/아키텍처 규칙 검사"`
- **model**: `sonnet`

리팩토링이 완료된 최종 코드에 lint를 실행하는 에이전트를 실행하라.

- `docs/CODE_RULES.md`를 읽고 `/lint`를 실행한다. (`./gradlew validateCode`)
- lint 실패 시 컨벤션에 맞게 수정한다. **최대 2회.**
- 로직 변경 금지. 스타일/컨벤션 수정만 허용.

### 8. 완료 처리

1. `phases/index.json` 업데이트:
   ```json
   {
     "status": "completed",
     "summary": "한 줄 요약 (다음 에이전트가 관련성 판단에 사용)",
     "completed_at": "ISO8601 타임스탬프"
   }
   ```
2. `phases/{slug}/context.md` 작성:
   ```markdown
   # {기능명}

   ## 생성/수정 파일
   - `src/main/java/.../controller/FooController.java`
   - `src/main/java/.../service/FooService.java`
   - `src/test/java/.../FooServiceTest.java`

   ## API 엔드포인트
   - `POST /api/v1/foo` — 요청/응답 구조 요약

   ## 핵심 인터페이스
   주요 메서드 시그니처

   ## 주요 결정
   - 결정 사항과 이유

   ## 주의사항
   - 이 기능에 의존할 때 알아야 할 것 (예: 사용하는 Filter, 공통 예외 클래스 등)

   ## 수동 확인
   ./gradlew test — 통과 확인
   (직접 눈으로 확인할 수 있는 방법)
   ```
3. `docs/LESSONS.md`에 이번 기능에서 발견한 패턴/실수가 있으면 추가한다.
4. `TODO.md`에서 해당 최상위 항목과 모든 하위 항목을 `[x]`로 업데이트한다.

### 9. 종료

```
✓ {기능명} 완료.
phases/{slug}/context.md 에서 구현 내용을 확인하세요.
다음 기능을 진행하려면 /harness 를 다시 실행하세요.
```

출력 후 **반드시 중단**한다. 다음 기능을 자동으로 시작하지 않는다.
