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
- 복잡한 비즈니스 로직이나 계산이 포함된 경우, 엣지 케이스(경계값, 캡/클램프, 0원 결제, 수량 분리 등)를 plan.md에 구체적으로 명시하여 테스트 에이전트가 누락 없이 반영할 수 있게 한다.
- 단, 테스트 시나리오는 묶을 수 있는 건 묶어 최소화한다.
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
- plan.md의 구현 범위와 엣지 케이스 목록에 있는 모든 케이스가 테스트로 존재해야 한다. 정상 케이스, 에러 케이스, 예외 처리, 엣지 케이스 포함.
- **`@ParameterizedTest` 사용 금지.** `@MethodSource`, `@ValueSource`, `@CsvSource`, `@NullAndEmptySource`, `name = "{0}"` 같은 파라미터화 패턴을 쓰지 않는다. 케이스마다 별도의 `@Test` 메서드로 작성해서, 어떤 입력으로 무엇을 검증하는지 메서드만 봐도 바로 알 수 있게 한다.
- **테스트 메서드명은 케이스를 그대로 설명하는 명확한 이름으로 짓는다.** 오버로드처럼 같은 이름을 재사용하거나 `test1`, `케이스2` 같은 모호한 이름 금지. `@DisplayName` 또는 한글 메서드명으로 "어떤 상황에서 어떤 결과가 나와야 하는지"를 명시한다. (예: `필수값이_누락되면_예외가_발생한다`, `조회_결과가_없으면_빈_목록을_반환한다`, `정상_요청이면_저장에_성공한다`)
- 테스트 케이스에 주석으로 "어떤 동작을 검증하는 테스트인지" 명시한다.
- 수동으로 확인할 수 있는 케이스는 주석에 샘플 요청/응답 데이터 포함.
- DB 검증이 필요한 경우 쉘에서 SQL을 직접 실행하지 않는다. 테스트 코드 내에서 Repository를 통해 검증한다.

테스트 파일이 하나도 없거나 plan.md 요구사항 대비 누락된 케이스가 있으면 이후 단계 진행 불가. 즉시 중단하고 사용자에게 보고한다.

**DB 컬럼 에러 처리:** 테스트 실행 중 DB NOT NULL 제약 위반 에러(예: `Column 'xxx' cannot be null`, `Field 'xxx' doesn't have a default value`)가 발생하면, 코드를 수정하지 말고 사용자에게 해당 컬럼명과 테이블명을 알려주고 DB 스키마 수정을 요청하라. 사용자가 수정 완료를 알리면 테스트를 재실행한다.

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
- **사람이 읽기 쉬운 가장 단순한 코드로 쓴다.** 쿼리를 한두 번 아끼려고 흐름을 꼬거나 분기·캐싱을 늘리지 않는다. 단, 반복문 안 단건 조회(N+1)처럼 건수에 비례해 늘어나는 쿼리는 일괄 조회한다.
  - 기존 메서드·흐름을 재사용한다. 한 곳에서만 쓰는 추상화, 일어날 수 없는 경우를 막는 방어 코드, 결과가 같은 중복 조건을 만들지 않는다.
  - 메서드 이름과 짧은 흐름만 봐도 무엇을 하는지 보이게 쪼갠다.
  - 주석은 큰 단계별 한 줄 주석 외에는 "왜"만 쓰고, 주변 코드의 주석 밀도에 맞춘다.
- **이름만 보고 무엇이 들었는지, 무엇을 하는지 알게 짓는다.** 새로 만들거나 이번에 고치는 코드에만 적용하고, 기존 이름을 일괄 개명하지 않는다.
  - 줄임말(`cnt`, `tmp`), 한 글자, 뭉뚱그린 이름(`data`, `result`, `map`)을 쓰지 않는다.
  - Map 은 `{값}By{키}`(`orderCountByUserId`), List·Set 은 담긴 것의 복수형(`userIds`).
  - boolean 은 `is/has/can` 으로 시작해 조건문이 문장처럼 읽히게 한다(`if (!hasPermission)`).
  - 메서드는 `동사 + 대상`(`process(order)` → `cancelExpiredOrder(order)`). `process`/`handle`/`execute` 같은 뭉뚱그린 동사는 쓰지 않는다.
  - 조건문 안의 계산식은 이름 붙은 지역 변수로 뺀다(`int totalCount = items.size() + newItems.size();`).
  - 같은 개념은 같은 이름으로 부르고, 도메인 용어는 엔티티·컬럼 이름을 따른다.
- **DTO 는 내부용이어도 `@Schema` 를 붙인다.** 클래스에 `@Schema(description)`, 모든 필드에 `@Schema(description, example)`. nullable 필드는 언제 null 인지 적고, 여러 줄이면 text block(`"""`)을 쓴다.

### 6. 리뷰 에이전트

Agent 툴로 실행하라:
- **name**: `"review-agent"`
- **description**: `"리뷰 에이전트 — {기능명} 코드 품질 검토"`
- **model**: `opus`

구현된 코드 전체와 테스트 파일을 검토하는 에이전트를 실행하라.

**검토 기준 (10년차 시니어 개발자 관점):**
- 클린 아키텍처 준수 여부
- 확장성과 유지보수성
- 코드 품질 (중복)
- 테스트가 plan.md 요구사항 기반으로 작성됐는지 (구현에 맞게 끼워 맞춘 테스트가 아닌지)
- **SQL/쿼리 효율성**: 호출 결과가 곧바로 무효화/덮어쓰여지는 불필요한 repository·service 호출이 없는지 확인. 단, 코드를 읽기 쉽게 하는 고정 1~2회 추가 쿼리는 지적하지 않는다.
  - 안티 패턴 예: 분기 조건이 이미 결정된 상태에서 "조회 후 null 로 덮어쓰기" (조회 자체를 스킵해야 함)
  - 안티 패턴 예: 같은 데이터를 같은 트랜잭션 내에서 여러 번 fetch — 한 번 조회해 지역 변수로 넘기되, 그러려고 메서드 인자가 늘거나 흐름이 꼬이면 각자 조회해도 된다
  - 안티 패턴 예: 반복문 안에서 단건 조회 (N+1) — `findAllByIdIn` / fetch join 으로 일괄 조회 검토
  - 안티 패턴 예: `exists` 면 충분한데 `findById` / `findAll` 로 엔티티 전체 로드
  - 발견 시 구체적 라인을 지적하고 "호출 자체 스킵" 또는 "일괄 조회" 등 개선 방향 제시
- **단순성·가독성**: 같은 동작을 더 적은 코드·클래스·분기로 쓸 수 있으면 지적한다. 예: 도달할 수 없는 분기, 한 번만 쓰는 추상화, 기존 메서드로 대체할 수 있는 새 구현, 중복 조건, "무엇"을 반복하는 주석, 쿼리를 아끼려고 복잡해진 코드. 단, plan.md 에서 사용자가 확정한 설계는 제외.
- **네이밍**: 줄임말·뭉뚱그린 변수명, 값이나 키가 빠진 Map 이름, 문장처럼 읽히지 않는 boolean, 조건문 안 계산식, `process`/`handle` 같은 메서드명을 지적하고 더 나은 이름을 제안한다. 이번 변경 범위 밖의 기존 이름은 제외.
- **DTO `@Schema`**: 이번에 만들거나 바꾼 DTO 는 내부용이어도 클래스 `@Schema(description)` 와 모든 필드 `@Schema(description, example)` 가 있는지 확인한다.

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

리팩토링이 완료된 최종 코드에 lint를 실행하는 에이전트를 실행하라. **이번 하네스에서 생성/변경한 파일만** 대상으로 한다.

**(A) 도구 검사 — `docs/CODE_RULES.md`의 "수동 실행 커맨드"를 읽고 실행한다.**
- **Spotless**: 변경된 파일 경로를 `-PspotlessFiles`로 한정해 검사한다.
  예: `./gradlew spotlessCheck -PspotlessFiles="src/main/java/.../FooService.java,..."`
- 프로젝트가 ArchUnit 등 추가 규칙을 CODE_RULES.md에 변경-파일-한정 커맨드로 정의해 뒀다면 그것도 함께 실행한다.
- 실패 시 컨벤션에 맞게 수정한다(Spotless 위반은 `spotlessApply`로 자동 수정). **최대 2회.**

**(B) 도구로 강제되지 않는 규칙 검토 — `docs/CODE_RULES.md`에 적혀 있지만 Spotless/ArchUnit으로 강제되지 않는 규칙들도 변경된 파일에 대해 직접 검토하고, 위반한 부분은 수정한다.** 이 규칙들은 빌드를 실패시키지 않으므로 도구 통과 여부와 무관하게 반드시 검토한다.
- API description(`@Operation`, `@Schema` 등)에 서버 내부 구현 용어(내부 메서드/클래스명, 분기·게이트·플래그 명칭, 내부 산식, 내부 문서 참조명, 리팩터링/버그 교정 이력 등)가 새어들지 않았는지 점검하고, 발견 시 `docs/CODE_RULES.md`의 "API description 작성 관점" 규칙대로 평이한 일반 언어로 풀어 쓰거나 코드 주석으로 옮긴다.

**공통:** 로직 변경 금지. 주석·스웨거 문서·포맷 등 스타일/컨벤션 수정만 허용.

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
