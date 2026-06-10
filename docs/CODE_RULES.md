# 코드 규칙

하네스 파이프라인의 Lint 에이전트가 기능 구현 마지막에 이 규칙을 기준으로 검사합니다.
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

## 3. 코드 작성 규칙

- CRITICAL: `@Autowired` 필드 주입 금지 — 생성자 주입만 허용하며, `@RequiredArgsConstructor` 우선 사용
- CRITICAL: 컨트롤러 public 매핑 메서드에 `@Operation(summary, description)` 필수, 파라미터에 `@Parameter(description)` 필수
- 주석은 한글로 작성한다
- 주요 로직 흐름에서 큰 단계별로 한 줄 한글 주석을 남긴다. 코드를 그대로 옮겨 적는 주석은 금지

### API 응답 변경 시 스웨거 업데이트

기존 API의 응답 필드가 추가되거나 의미가 변경되면, 해당 컨트롤러 메서드의 `@Operation(description)`에 변경 내용을 명시한다. FO 개발자가 스웨거만 보고 파악할 수 있어야 한다.

### @Operation description 작성

`@Operation(description = ...)`이 여러 줄이거나 줄바꿈/마크다운을 포함하면 **Java text block(`"""..."""`)**을 사용한다. 문자열 연결(`"..." + "\n\n" + "..."`) 금지 — 가독성과 스웨거 렌더링 일관성 확보 목적.

```java
@Operation(summary = "게시글 목록 조회", description = """
    카테고리별 게시글 목록을 조회합니다.

    ### [응답 필드]
    - `commentCount` — 게시글별 댓글 수
    - `pinned` (nullable) — 고정 게시글 여부, 일반 게시글이면 null
    """)
@GetMapping("/posts")
public ResponseEntity<ResponseDTO<PostListResponse>> getPosts(
        @Parameter(description = "카테고리 ID") @RequestParam Long categoryId,
        @Parameter(description = "페이지 커서") @RequestParam(required = false) Long cursor,
        @Parameter(description = "페이지 크기") @RequestParam(defaultValue = "20") int size) {
    ...
}
```

### API description 작성 관점 (소비자 관점 / 내부 구현 노출 금지)

`@Operation`, `@Schema` 등의 description을 읽는 대상은 **API 소비자(프론트엔드)**다. 따라서 description은 소비자 관점에서 "무엇을 받고, 무엇을 그리면 되는지"만 기술하고, **서버 내부 구현 디테일은 노출하지 않는다.**

- CRITICAL: API description은 **API 소비자(프론트엔드) 관점**으로 작성한다.
- 다음과 같은 **서버 내부 구현 디테일은 description에 노출 금지**:
  - 내부 메서드/클래스/컴포넌트 이름 (예: `resolveXxx`, `XxxValidator`)
  - 내부 분기·게이트·플래그 로직의 명칭이나 동작 설명
  - 내부 알고리즘·산식 (반올림/절삭 공식, 가중치 계산식 등)
  - 내부 문서 참조명 (예: "계산 방식 B", "정책 v2 표 3")
  - 리팩터링/버그 교정 이력 (예: "기존 X 버그를 Y로 교정")
- 위 내용이 소비자의 동작에 영향을 주면, 내부 용어/공식 대신 **평이한 일반 언어로 풀어서** 적는다. 필요하면 `> 참고:` / `[참고]` 같은 보조 노트 형태로 남긴다.
  - 예) 내부 정렬 산식·가중치 노출(❌) → "목록은 정렬된 순서로 내려오므로 소비자는 별도 가공 없이 받은 순서대로 그리면 됩니다"(⭕)
  - 예) 내부 필터 분기 설명(❌) → "일부 항목은 서버가 목록에서 자동 제외하므로 소비자는 받은 목록을 그대로 그리면 됩니다"(⭕)
  - 예) 내부 게이트 명칭(❌) → "특정 조건에서는 빈 목록/`null`이 반환됩니다"(⭕)
  - 위 ⭕ 문구는 **변환 방향을 보여주는 패턴**이며 그대로 복붙하는 템플릿이 아니다. 실제 작성 시 "특정 조건" 같은 placeholder는 해당 API의 실제 정책으로 채우되, 그 조건 역시 내부 게이트/플래그 명칭이 아니라 소비자가 이해할 수 있는 말로 적는다 (예: "비공개 상태인 대상은 `null`이 반환됩니다", "조회 권한이 없으면 빈 목록이 반환됩니다").
- 순수 내부 로직 설명은 description이 아니라 **코드 주석**에 둔다.

### 주석 컨벤션

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
