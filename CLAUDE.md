# 프로젝트: {프로젝트명}

## 기술 스택
- {프레임워크 (예: Next.js 15)}
- {언어 (예: TypeScript strict mode)}
- {스타일링 (예: Tailwind CSS)}

## 아키텍처 규칙
- CRITICAL: {절대 지켜야 할 규칙 1 (예: 모든 API 로직은 app/api/ 라우트 핸들러에서만 처리)}
- CRITICAL: {절대 지켜야 할 규칙 2 (예: 클라이언트 컴포넌트에서 직접 외부 API를 호출하지 말 것)}
- {일반 규칙 (예: 컴포넌트는 components/ 폴더에, 타입은 types/ 폴더에 분리)}
- 컨트롤러 public 매핑 메서드는 `@Operation(summary, description)` 필수. 파라미터에 `@Parameter(description)` 필수
- API description(`@Operation`, `@Schema` 등)은 API 소비자(프론트엔드) 관점으로 작성한다. 내부 메서드/클래스명·분기 로직·산식 등 서버 내부 구현 디테일은 노출하지 말고 평이한 일반 언어로 풀어 쓰며, 순수 내부 로직 설명은 코드 주석에 둔다
- 주석은 한글로 작성한다
- 주요 로직 흐름에서 큰 단계별로 한 줄 한글 주석을 남긴다. 코드를 그대로 옮겨 적는 주석은 금지
- `@Autowired` 필드 주입 금지, 생성자 주입만 허용. 생성자는 `@RequiredArgsConstructor` 우선 사용

## 개발 프로세스
- CRITICAL: 새 기능 구현 시 반드시 테스트를 먼저 작성하고, 테스트가 통과하는 구현을 작성할 것 (TDD)
- 커밋 메시지는 conventional commits 형식을 따를 것 (feat:, fix:, docs:, refactor:)

## 명령어
npm run dev      # 개발 서버
npm run build    # 프로덕션 빌드
npm run lint     # ESLint
npm run test     # 테스트
