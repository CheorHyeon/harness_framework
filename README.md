# Harness Framework

Claude Code 스킬 기반의 기능 단위 개발 프레임워크입니다.
개발자가 TODO를 작성하면 Claude가 Plan → 테스트 → 구현 → 리뷰 → Lint 파이프라인을 자동으로 실행합니다.

---

## 시작하는 법

### 신규 프로젝트

1. `CLAUDE.md`에 기술 스택, 아키텍처 규칙, 도메인 참조 파일 경로를 작성하세요.
2. `docs/PROJECT.md`에 프로젝트 설명, 대상 사용자, 배경을 작성하세요.
3. `docs/CODE_RULES.md`에 포맷/아키텍처 규칙을 작성하세요.
4. `TODO.md`에 구현할 기능 목록을 작성하세요.
5. `/harness`를 실행하세요.

### 기존 프로젝트 도입

1. `CLAUDE.md`에 기존 코드베이스의 기술 스택, 아키텍처 규칙을 작성하세요.
2. `docs/PROJECT.md`에 프로젝트 맥락을 작성하세요.
3. `docs/CODE_RULES.md`에 기존 프로젝트의 코딩 컨벤션, 포맷/아키텍처 규칙을 작성하세요. pre-commit hook이 없으므로 Lint 에이전트가 기능 구현 마지막에 이 규칙을 기준으로 검사합니다.
4. `TODO.md`에 남은 기능 목록을 작성하세요.
5. `/harness`를 실행하세요.

### 다른 기술 스택에 적용

기본 설정은 Java/Spring Boot 백엔드 기준으로 작성되어 있습니다. 다른 기술 스택에 적용하려면 Claude에게 "기술 스택을 X로 바꿔줘"라고 요청하세요.

---

## 워크플로우

```
개발자: TODO.md에 기능 목록 작성
  ↓
/harness 실행
  ↓
Plan 에이전트 → plan.md 작성
  ↓
개발자가 plan.md 검토/승인  ← 최소 3-4번 피드백 권장
  ↓
테스트 에이전트 → plan.md 기반 통합 테스트 작성 (케이스 누락 시 중단)
  ↓
구현 에이전트 → 테스트 통과하는 구현 (테스트 파일 수정 금지)
  ↓
리뷰 에이전트 (시니어 관점, 최대 3회 재시도)
  ↓
Lint 에이전트 → ./gradlew validateCode
  ↓
완료 → context.md 작성 → 멈춤
  ↓
개발자가 직접 검증
  ↓
/harness 재실행 → 다음 기능
```

---

## 파일 구조

```
TODO.md                    # 개발자 작성: 구현해야 하는 기능 목록
CLAUDE.md                  # 기술 스택, 아키텍처 규칙, 도메인 참조 파일 경로
docs/
  PROJECT.md               # 개발자 작성: 프로젝트 설명, 대상 사용자, 제품 배경
  LESSONS.md               # 에이전트 자동 누적: 구현 중 발견한 실수/패턴
  CODE_RULES.md            # 개발자 작성: 포맷/아키텍처 규칙 + 수동 실행 커맨드
.claude/
  settings.json            # PreToolUse hook: 위험 명령어 자동 차단
  commands/
    harness.md             # /harness 스킬
    lint.md                # /lint 스킬
    ask.md                 # /ask 스킬
phases/
  index.json               # 전체 진행 상황 + 기능별 1줄 summary
  {feature-slug}/          # 영문 kebab-case
    plan.md                # Plan 에이전트 출력 (사용자 승인 후 구현 시작)
    review-feedback.md     # 리뷰 에이전트 출력 (불통과 시 구현 에이전트 참고 문서 + 개발자 참고용)
    context.md             # 완료 후: 구현 요약 + 수동 확인 방법
```

### TODO.md 형식

```markdown
# TODO

- [ ] 로그인 기능 구현
  - [ ] 이메일/비밀번호 회원가입
  - [ ] 로그인 → JWT 발급

- [ ] 유저 프로필
  - [ ] 프로필 조회
  - [ ] 프로필 수정
```

최상위 `- [ ]` 항목 1개 = `/harness` 1회 실행 단위. 하위 항목은 Plan 에이전트가 구현 범위 파악에 활용합니다.

---

## 커맨드

### `/harness`

다음 미완료 기능을 Plan → 테스트 → 구현 → 리뷰 → Lint 파이프라인으로 처리합니다.
기능 1개 완료 후 반드시 멈추고, 사람이 검증 후 다시 실행해야 합니다.

### `/lint`

포맷(Spotless) + 아키텍처(ArchUnit) 규칙을 검사하고 위반 항목을 안내합니다.
규칙 상세는 `docs/CODE_RULES.md`를 참조하세요. 하네스 파이프라인의 Lint 에이전트도 이 커맨드를 사용합니다.

> 기본 설정은 전체 파일을 검사합니다. 하네스 실행 중 생성/수정된 파일만 검사하고 싶다면 Claude에게 "Lint 검사를 변경된 파일만 하도록 바꿔줘"라고 요청하세요.

### `/ask {slug} {질문}`

완료된 기능의 구현 내용을 질문합니다. slug는 `phases/index.json`에서 확인할 수 있습니다.

```
/ask login-feature "JwtAuthenticationFilter는 어디서 등록돼?"
```

context.md와 소스 파일을 읽고 답변합니다. 세션이 끊긴 후에도 사용 가능합니다.

---

## error 기능 재시도

구현 중 리뷰 3회 실패 시 `index.json`에 `status: error`와 `error_summary`가 기록됩니다.

```
/harness 재실행
  ↓
index.json에서 error 상태 기능 발견
  ↓
Plan 에이전트가 error_summary로 원인 분석
  ↓
  [API 키, 환경변수 등 사용자 개입이 필요한 경우]
  → 사용자에게 확인 요청 → 답변 반영
  [코드 수준 문제인 경우]
  → 자동으로 재시도
  ↓
plan.md 재작성 → 파이프라인 재시작
```

---

## 안전장치

`.claude/settings.json`의 PreToolUse hook이 아래 명령을 자동 차단합니다:

| 차단 패턴 | 이유 |
|---|---|
| `rm -rf` | 파일 대량 삭제 |
| `git push --force` | 원격 브랜치 강제 덮어쓰기 |
| `git reset --hard` | 로컬 변경사항 전체 폐기 |
| `DROP TABLE` | DB 테이블 삭제 |

---

## 피드백 루프

구현 중 발견한 실수나 패턴은 아래 파일에 기록되어 다음 기능에 반영됩니다:

| 실수 유형 | 기록 위치 |
|---|---|
| 아키텍처/절대 규칙 위반 | `CLAUDE.md` CRITICAL 섹션 |
| 반복 가능한 구현 패턴 실수 | `docs/LESSONS.md` |

---

## 그냥 Claude Code에 시키는 것과 차이

| | Claude Code만 쓸 때 | Harness |
|---|---|---|
| TDD 강제 | "테스트 먼저 써줘" 요청 (구현에 맞게 끼워 맞출 수 있음) | 테스트/구현 에이전트 분리 + plan.md 요구사항 기반 케이스 커버리지 강제 |
| Plan 검토 | Plan 모드 직접 사용 | 자동으로 Plan → 승인 단계 포함 |
| 세션 간 지식 유지 | CLAUDE.md + memory 직접 설정 | context.md + LESSONS.md 자동 누적 |

결국 목적은 **편의성**입니다. 매번 직접 챙겨야 할 것들(Plan 검토, TDD, 리뷰, Lint)을 파이프라인으로 자동화한 것입니다. 이미 이 흐름을 직접 챙기는 분이라면 큰 차이는 없습니다.

---

## 기존 방식의 아쉬운 점

이전 버전은 Python(`scripts/execute.py`)이 `claude -p` 서브프로세스를 반복 호출하는 방식이었습니다.

**실행 구조가 phase > step 2단계**였고, 모든 step에 docs 전체가 강제로 주입되었습니다:

```
phase 1 (로그인 기능):
  step0 → [CLAUDE.md + docs/*.md 전부 + step0.md] → 완료
  step1 → [CLAUDE.md + docs/*.md 전부 + step1.md] → 완료
  step2 → [CLAUDE.md + docs/*.md 전부 + step2.md] → 완료
  step3 → [CLAUDE.md + docs/*.md 전부 + step3.md] → 완료
  ↓ 자동으로 다음 phase
phase 2 (유저 프로필):
  step0 → [CLAUDE.md + docs/*.md 전부 + step0.md] → ...
```

- **에이전틱하지 않음** — Python이 무엇을 줄지 미리 결정해서 텍스트로 밀어넣음. 에이전트가 스스로 필요한 파일을 선택하지 못함
- **사람 개입 불가** — 실행 시작 후 모든 기능이 끝날 때까지 멈추지 않음. 로그인이 잘못 구현돼도 결제까지 완료된 후에야 발견
- **docs 반복 주입** — 기능 1개에 step이 4개면 docs가 4번 주입. 관련 없는 내용도 무조건 포함

| | 이전 | 이후 |
|---|---|---|
| 오케스트레이터 | Python (~400줄) | Claude 스킬 파일 (md) |
| 에이전트 실행 | `claude -p` 서브프로세스 | Agent 툴 (서브에이전트) |
| 실행 단위 | phase > step | 기능 1개 |
| 사람 개입 | 실행 시작 후 없음 | 기능마다 plan 승인 + 완료 후 검증 |
| docs 주입 | 모든 step에 전체 주입 | Plan 에이전트만 읽음 |
| TDD | 파일 존재 여부만 체크 | plan.md 요구사항 기반 통합 테스트 작성 → 구현이 통과시켜야 진행 |
