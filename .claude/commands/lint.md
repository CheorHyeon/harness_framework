# /lint

포맷 + 아키텍처 규칙을 검사하고 위반 항목을 안내합니다.
규칙 상세는 `docs/CODE_RULES.md`를 참조하세요.

## 실행 순서

1. `./gradlew validateCode` 실행
2. 성공하면 "lint 통과" 출력 후 종료
3. 실패하면 위반 항목을 분석해서 아래 형식으로 출력:

```
[Spotless] 포맷 위반
- src/main/java/.../FooController.java: 미사용 import, 줄 끝 공백
→ 수정: ./gradlew spotlessApply -PspotlessFiles="..."

[ArchUnit] 아키텍처 위반
- FooController.getFoo(): @Operation 어노테이션 누락
- BarException: ApplicationException 상속 누락
→ 수정: 위 항목을 직접 수정 후 재실행
```

4. 위반 항목을 수정합니다.
   - Spotless 위반은 `spotlessApply`로 자동 수정
   - ArchUnit 위반은 직접 코드 수정
5. 수정 후 `./gradlew validateCode` 재실행해서 통과 확인
