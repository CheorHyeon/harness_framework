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
| 1 | 컨트롤러 반환 타입 | 컨트롤러 매핑 메서드는 `ResponseEntity<T>` 반환 필수 (void인 경우 `ResponseEntity<Void>`) |
| 2 | Swagger 어노테이션 | 컨트롤러 public 매핑 메서드는 `@Operation(summary, description)` 필수. 파라미터에 `@Parameter(description)` 필수 |
| 3 | DTO `@Schema` | DTO 필드에 `@Schema(description = "한글 설명", example = "예시값")` 필수. description은 한글로 작성 |
| 4 | `@RequestMapping` 경로 | 컨트롤러 경로는 `/api/v1`, `/api/v2`, `/api/v3` 중 하나로 시작 필수 |
| 5 | 레이어 의존성 방향 | `Controller → Service → Repository` 단방향만 허용. 역방향 의존 금지 |
| 6 | 클래스 네이밍 접미사 | Controller/Service/Repository 클래스는 각각 해당 접미사 필수 |
| 7 | 필드 주입 금지 | `@Autowired` 필드 주입 금지, 생성자 주입만 허용. 생성자는 `@RequiredArgsConstructor` 우선 사용 |

---

## 3. 주석 컨벤션

- 주석은 한글로 작성한다.
- 주요 로직 흐름에서 큰 단계별로 한 줄 주석을 남긴다. 코드를 그대로 옮겨 적는 주석은 금지.

**좋은 예:**
```java
public PostListResponse getPosts(Long cursor, int size) {
    // size 보정 (1~50 범위)
    int correctedSize = correctSize(size);

    // 게시글 목록 조회 (size+1개로 다음 페이지 존재 여부 판단)
    List<Post> posts = fetchPosts(cursor, correctedSize);

    // 배치 조회로 연관 데이터 로드
    List<Long> postIds = posts.stream().map(Post::getId).toList();
    Map<Long, List<Comment>> commentMap = commentRepository.findByPostIds(postIds);

    // 응답 DTO 조합
    return PostListResponse.of(posts, commentMap, correctedSize);
}
```

**나쁜 예:**
```java
// correctedSize에 correctSize 결과를 할당한다
int correctedSize = correctSize(size);
// posts 변수에 fetchPosts 결과를 넣는다
List<Post> posts = fetchPosts(cursor, correctedSize);
```

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
