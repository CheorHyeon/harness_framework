# 코드 규칙

커밋 전 pre-commit hook으로 자동 검사되는 규칙입니다.
변경된 Java 파일이 없으면 검사 전체가 스킵됩니다.

---

## 1. Spotless 코드 포맷

변경된 파일만 대상으로 실행됩니다.

**Java 파일:**
- 미사용 import 제거 (`removeUnusedImports`)
- 줄 끝 공백 제거 (`trimTrailingWhitespace`)
- 파일 끝 개행 추가 (`endWithNewline`)

**기타 파일 (`.yml`, `.yaml`, `.md`, `.gradle`):**
- 줄 끝 공백 제거
- 파일 끝 개행 추가

---

## 2. ArchUnit 아키텍처 규칙

`src/main/java` 경로 변경이 있을 때만 실행됩니다.

| # | 규칙 | 설명 |
|---|---|---|
| 1 | 예외 클래스 | `..exception..` 패키지의 모든 예외는 `ApplicationException` 상속 필수 |
| 2 | 컨트롤러 반환 타입 | `@GetMapping` 등 매핑 메서드는 `ResponseDTO<T>` 또는 `ResponseEntity<ResponseDTO<T>>` 반환 필수 |
| 3 | Swagger 어노테이션 | 컨트롤러 public 매핑 메서드는 `@Operation` 필수 |
| 4 | DTO `@Schema` | `io.travelesim.domain..dto..` 패키지 클래스는 `@Schema` 필수 (클래스 레벨 또는 모든 필드) |
| 5 | `@RequestMapping` 경로 | 컨트롤러 경로는 `/api/v1`, `/api/v2`, `/api/v3`, `/api/test` 중 하나로 시작 필수 |
| 6 | 테스트 API 프로파일 | `/api/test` 사용 시 `@Profile({"dev", "local"})` 필수 |
| 7 | `log.error()` 호출 | `GlobalExceptionRestAdvice` 제외, `log.error()` 호출 시 `ExceptionLogPrefix` enum 참조 필수 |

---

## 수동 실행 커맨드

```bash
# 포맷 검사만
./gradlew spotlessCheck

# 포맷 자동 수정
./gradlew spotlessApply

# 특정 파일만 포맷 수정
./gradlew spotlessApply -PspotlessFiles="경로/파일.java"

# 아키텍처 규칙 전체 검사
./gradlew verificationTest

# 특정 클래스만 아키텍처 검사
./gradlew verificationPartial -Parchunit.includes="io.travelesim.domain.foo.controller.FooController"

# 포맷 + 아키텍처 통합 검사
./gradlew validateCode
```
