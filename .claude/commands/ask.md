구현된 기능에 대해 질문에 답변한다.

## 사용법

```
/ask {slug} {질문}
예) /ask login-feature "JWT 만료 처리는 어떻게 되어있어?"
```

slug를 모를 경우 `phases/index.json`을 읽어 `name` 필드로 찾는다.

## 절차

1. `phases/index.json`을 읽어 해당 slug의 기능을 확인한다.
   - slug가 없으면 "해당 기능을 찾을 수 없습니다. phases/index.json에서 slug를 확인하세요." 출력 후 종료.
   - status가 `pending`이면 "아직 구현되지 않은 기능입니다." 출력 후 종료.
   - status가 `error`이면 "구현이 완료되지 않았습니다. error_summary: {error_summary 내용}" 출력 후 종료.

2. `phases/{slug}/context.md`를 읽는다.

3. context.md의 "생성/수정 파일" 목록에 있는 소스 파일들을 읽는다.

4. 위 내용을 바탕으로 질문에 답변한다.
