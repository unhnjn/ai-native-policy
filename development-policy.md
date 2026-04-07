# Development Policy
## 사내 작업 관리 도구 — Todo List

> 코드 작성, 리뷰, 브랜치 관리, 테스트에 관한 팀 규칙.
> 새 기여자가 합류하면 이 문서부터 읽는다.

---

## 1. 기술 스택 & 버전

| 레이어 | 기술 | 버전 |
|--------|------|------|
| 런타임 | Node.js | 20 LTS |
| 백엔드 언어 | TypeScript (strict) | 5.x |
| 백엔드 프레임워크 | Express | 4.x |
| ORM | Prisma | 5.x |
| DB (개발/데모) | SQLite | - |
| DB (운영) | PostgreSQL | 15+ |
| 프론트엔드 언어 | TypeScript (strict) | 5.x |
| 프론트엔드 프레임워크 | React | 18.x |
| 번들러 | Vite | 5.x |
| 스타일링 | Tailwind CSS | 3.x |
| 서버 상태 | TanStack Query | 5.x |
| HTTP 클라이언트 | Axios | 1.x |
| 아이콘 | lucide-react | 최신 |
| 유효성 검증 | Zod | 3.x |

**버전 고정 원칙**: `package.json`의 의존성은 `^` 허용. 단, Prisma 관련 패키지는 `prisma`와 `@prisma/client`의 버전을 항상 일치시킨다.

---

## 2. 프로젝트 구조 규칙

```
ai-native-demo/
├── backend/
│   ├── prisma/schema.prisma     # 단일 스키마 파일 — 여기서만 DB 구조 변경
│   └── src/
│       ├── routes/              # 라우트 파일 = 엔티티 단위 (todos, auth, categories, tags, trash)
│       ├── services/            # 라우트와 분리된 비즈니스 로직 (stateTransition, reminderWorker)
│       ├── middleware/          # Express 미들웨어 (auth)
│       └── lib/                 # 공유 인스턴스 (prisma client singleton)
├── frontend/
│   └── src/
│       ├── api/client.ts        # Axios 인스턴스 단 하나 — 직접 axios import 금지
│       ├── hooks/               # TanStack Query 훅 — 서버 상태 접근은 반드시 훅을 통해
│       ├── components/          # 재사용 UI 컴포넌트 (페이지 전용 컴포넌트는 pages/ 안에)
│       ├── pages/               # 라우트 단위 페이지 컴포넌트
│       └── types/index.ts       # 공유 타입 정의 — 중복 타입 선언 금지
└── docs/                        # 정책 및 스펙 문서
```

### 금지 패턴

- 라우트 핸들러 안에 비즈니스 로직 직접 작성 → `services/`로 추출
- 컴포넌트 안에서 `axios` 직접 호출 → `hooks/` 경유 필수
- `any` 타입 사용 → 최소한 `unknown`으로, 가능하면 명시적 타입
- `console.log` 를 디버그 용도로 커밋 → 리마인더 워커의 mock 이메일 출력은 예외

---

## 3. 백엔드 코딩 컨벤션

### 라우트 구조 패턴

```typescript
// 1. 스키마 정의 (Zod) — 라우트 파일 상단
const CreateXxxSchema = z.object({ ... })

// 2. 라우트 핸들러 — 반환 타입은 Promise<void>
router.post("/", async (req: Request, res: Response): Promise<void> => {
  // 3. 유효성 검증 먼저
  const result = CreateXxxSchema.safeParse(req.body)
  if (!result.success) {
    res.status(400).json({ error: "Validation failed", issues: result.error.issues })
    return
  }
  // 4. 소유권 확인 (다른 유저 리소스 접근 차단)
  // 5. 비즈니스 로직
  // 6. try/catch — 에러는 반드시 console.error + 500 응답
})
```

### 에러 응답 형식 (반드시 준수)

```json
{ "error": "에러 설명 (영문 권장)", "issues": [] }
```

| 상황 | 코드 |
|------|------|
| 유효성 오류 | 400 |
| 인증 없음 | 401 |
| 타인 리소스 접근 | 403 |
| 리소스 없음 | 404 |
| 상태 전이 오류 | 422 |
| 서버 내부 오류 | 500 |

### 상태 전이 규칙 변경

`src/services/stateTransition.ts`의 `ALLOWED_TRANSITIONS` 객체만 수정한다.
라우트 핸들러에서 직접 전이 조건을 검사하지 않는다.

### Prisma 사용 규칙

- `prisma` 인스턴스는 `src/lib/prisma.ts`의 싱글턴만 사용
- 연관 데이터가 필요한 조회는 `include`로 단일 쿼리 처리 (N+1 금지)
- 다중 쓰기가 필요하면 `prisma.$transaction()` 사용
- 스키마 변경 시: `prisma db push` (개발) → `prisma migrate dev` (운영 배포 전)

---

## 4. 프론트엔드 코딩 컨벤션

### 서버 상태 관리 패턴

```typescript
// hooks/useXxx.ts — 모든 API 통신은 여기서
export function useCreateXxx() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: async (payload) => { ... },
    onMutate: async (payload) => {
      // 1. 진행 중인 쿼리 취소
      await queryClient.cancelQueries({ queryKey: ['xxx'] })
      // 2. 현재 상태 스냅샷
      const previous = queryClient.getQueriesData(...)
      // 3. Optimistic 업데이트
      queryClient.setQueriesData(...)
      return { previous }
    },
    onError: (_err, _vars, context) => {
      // 4. 롤백
      for (const [key, data] of context.previous) {
        queryClient.setQueryData(key, data)
      }
      toast.error('요청을 처리하지 못했습니다. 다시 시도해주세요.')
    },
    onSettled: () => {
      // 5. 서버와 재동기화
      queryClient.invalidateQueries({ queryKey: ['xxx'] })
    },
  })
}
```

**Optimistic UI 필수 적용 대상**: 상태 변경, 제목 수정, 삭제, 우선순위 변경
**적용 불필요**: 복원, 카테고리 CRUD, 태그 CRUD (빈도 낮고 롤백 UX 복잡)

### 컴포넌트 작성 규칙

- 컴포넌트 파일 1개 = 컴포넌트 1개 (named export 금지, default export 사용)
- Props 타입은 컴포넌트 파일 안에 인라인으로 선언 (재사용되면 `types/index.ts`로 이동)
- 이벤트 핸들러 이름: `handle` 접두사 (`handleClick`, `handleSubmit`)
- 조건부 렌더링: 삼항 연산자는 단일 표현식만, 복잡한 분기는 변수로 추출

### Query Key 규칙

```typescript
['todos']                    // 할일 전체 목록 (필터 무관)
['todos', filters]           // 필터 적용 목록
['todos', 'trash']           // 휴지통
['categories']               // 카테고리 목록
['tags']                     // 태그 목록
```

Query key 변경 시 `invalidateQueries` 호출부도 반드시 같이 수정한다.

---

## 5. 타입 정의 규칙

- 프론트엔드 API 응답 타입 → `frontend/src/types/index.ts`
- 백엔드 Zod 스키마 → 각 라우트 파일 내부
- 백엔드-프론트엔드 간 타입 공유 필요 시 → 별도 `shared/` 패키지로 추출 (현재 미구성)

### 핵심 타입

```typescript
// Todo 상태 — 백엔드 stateTransition.ts와 반드시 동기화
type TodoStatus = "TODO" | "IN_PROGRESS" | "DONE" | "DELETED"

// 우선순위
type Priority = "HIGH" | "MEDIUM" | "LOW"
```

**주의**: 백엔드 스키마를 SQLite String 타입으로 변경했으므로 Prisma enum이 없다.
상태값 오타 방지를 위해 위 타입을 항상 참조한다.

---

## 6. 환경변수 관리

### 백엔드 (`backend/.env`)

| 변수 | 설명 | 기본값 |
|------|------|--------|
| `DATABASE_URL` | DB 연결 문자열 | `file:./dev.db` (SQLite) |
| `JWT_SECRET` | Access Token 서명 키 | 필수, 최소 32자 |
| `JWT_REFRESH_SECRET` | Refresh Token 서명 키 | 필수, 최소 32자 |
| `JWT_EXPIRES_IN` | Access Token 만료 | `15m` |
| `JWT_REFRESH_EXPIRES_IN` | Refresh Token 만료 | `7d` |
| `PORT` | 서버 포트 | `4000` |
| `CORS_ORIGIN` | 허용 Origin | `*` (개발), 운영 시 명시 필수 |

### 프론트엔드 (`frontend/.env`)

| 변수 | 설명 | 기본값 |
|------|------|--------|
| `VITE_API_URL` | 백엔드 API URL | `http://localhost:4000` |

**규칙**:
- `.env` 파일은 절대 커밋하지 않는다 (`.gitignore` 확인)
- `.env.example`에 키 목록과 설명을 항상 최신 상태로 유지한다
- 시크릿 키는 운영에서 최소 64자 랜덤 문자열 사용

---

## 7. 브랜치 & 커밋 전략

### 브랜치 네이밍

```
feature/todo-drag-and-drop      # 새 기능
fix/status-transition-422       # 버그 수정
chore/upgrade-prisma-5-x        # 의존성·설정 변경
docs/update-dev-policy          # 문서 수정
```

### 커밋 메시지 (Conventional Commits)

```
feat: 할일 드래그&드롭 정렬 추가
fix: DONE→DELETED 전이 허용 오류 수정
chore: Prisma 5.22 → 5.23 업그레이드
docs: development-policy 환경변수 섹션 추가
refactor: reminderWorker 재시도 로직 분리
test: useTodos 낙관적 업데이트 롤백 테스트 추가
```

### PR 체크리스트

- [ ] `npx tsc --noEmit` 통과 (백엔드, 프론트엔드 모두)
- [ ] 새 API 엔드포인트에 `requireAuth` 미들웨어 적용 확인
- [ ] 새 DB 컬럼/테이블에 `prisma migrate dev` 마이그레이션 파일 포함
- [ ] 상태 전이 규칙 변경 시 `stateTransition.ts` + `feature-policy.md` 동시 업데이트
- [ ] 환경변수 추가 시 `.env.example` 업데이트

---

## 8. 테스트 정책

### 현재 상태 (v1)
자동화 테스트 미구성. 수동 API 테스트로 검증.

### 추가 목표 (v2)

**백엔드**
- 테스트 프레임워크: Vitest + Supertest
- 필수 테스트 대상:
  - `stateTransition.ts` — 모든 전이 케이스 (허용/거부)
  - `POST /todos` — 유효성 검증 케이스
  - `PATCH /todos/:id` — 상태 전이 사이드 이펙트 (리마인더 생성/취소)
  - `requireAuth` 미들웨어 — 토큰 없음/만료/유효

**프론트엔드**
- 테스트 프레임워크: Vitest + Testing Library
- 필수 테스트 대상:
  - `useTodos` — Optimistic 업데이트 롤백
  - `TodoCard` — 인라인 편집 Enter/Esc 동작
  - `FilterBar` — 다중 선택 AND 조합

**커버리지 목표**: 비즈니스 로직 (services/, hooks/) 80% 이상

---

## 9. 로컬 개발 환경 셋업

```bash
# 1. 루트 의존성 설치
npm install

# 2. 백엔드 의존성 + DB 초기화
cd backend
npm install
cp .env.example .env        # 필요 시 DATABASE_URL 등 수정
npx prisma db push          # SQLite dev.db 생성 + 스키마 동기화
npx prisma generate         # Prisma Client 재생성

# 3. 프론트엔드 의존성
cd ../frontend
npm install

# 4. 동시 실행 (루트)
cd ..
npm run dev
# 백엔드: http://localhost:4000
# 프론트엔드: http://localhost:3000
```
