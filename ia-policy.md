# IA & 메뉴 구조 정책 — 사내 작업 관리 도구

> 버전: 1.0.0
> 최종 수정일: 2026-04-07
> 작성 대상: 프론트엔드 개발자, UI/UX 디자이너, 기획자

---

## 목차

1. [전체 IA 트리](#1-전체-ia-트리)
2. [라우팅 구조](#2-라우팅-구조)
3. [컴포넌트 계층 트리](#3-컴포넌트-계층-트리)
4. [네비게이션 플로우](#4-네비게이션-플로우)
5. [URL 설계 규칙](#5-url-설계-규칙)
6. [상태별 화면 분기](#6-상태별-화면-분기)

---

## 1. 전체 IA 트리

앱의 전체 정보 구조를 트리 형태로 표현한다. 인증 여부에 따라 진입 가능한 영역이 분리된다.

```
앱 (Todo App)
├── 인증 영역 (비로그인)
│   └── /login
│       ├── 이메일 입력
│       ├── 이름 입력
│       └── 시작하기 버튼
└── 인증 영역 (로그인)
    ├── 글로벌 헤더
    │   ├── 로고/앱명
    │   ├── 네비게이션 링크 (할일 목록, 휴지통)
    │   ├── 단축키 힌트 배지
    │   └── 로그아웃 버튼
    ├── /todos (메인 목록)
    │   ├── 필터 바
    │   │   ├── 상태 필터 (다중 선택 토글)
    │   │   │   ├── TODO 토글
    │   │   │   ├── IN_PROGRESS 토글
    │   │   │   └── DONE 토글
    │   │   ├── 우선순위 필터 (다중 선택 토글)
    │   │   │   ├── HIGH 토글
    │   │   │   ├── MEDIUM 토글
    │   │   │   └── LOW 토글
    │   │   ├── 카테고리 필터 (드롭다운 멀티셀렉트)
    │   │   └── 필터 초기화 버튼
    │   ├── 할일 리스트
    │   │   └── TodoCard (반복)
    │   │       ├── 체크박스 (완료 토글)
    │   │       ├── 제목 (인라인 편집)
    │   │       ├── 우선순위 아이콘 (클릭 → PriorityDropdown)
    │   │       ├── 상태 뱃지
    │   │       ├── 마감일 표시
    │   │       ├── 카테고리 칩
    │   │       ├── 태그 칩들
    │   │       └── 삭제 버튼 (클릭 → ConfirmDialog)
    │   ├── 빈 상태 (Empty State)
    │   │   ├── 일러스트/아이콘
    │   │   ├── 안내 문구 (첫 사용 / 필터 결과 없음 분기)
    │   │   └── 새 할일 만들기 버튼 (첫 사용 시만 노출)
    │   └── FAB / 새 할일 버튼
    │       └── 클릭 → TodoModal (생성)
    ├── /trash (휴지통)
    │   ├── 안내 문구 (30일 자동 삭제 안내)
    │   └── 삭제 할일 리스트
    │       └── TrashCard (반복)
    │           ├── 제목
    │           ├── 삭제일 / 남은 일수 뱃지
    │           ├── 복원 버튼
    │           └── 영구 삭제 버튼 (클릭 → ConfirmDialog)
    └── 모달/오버레이 레이어
        ├── TodoModal (할일 생성)
        │   ├── 제목 입력 (필수)
        │   ├── 설명 입력 (선택)
        │   ├── 마감일 DatePicker (선택)
        │   ├── 우선순위 선택 (기본: MEDIUM)
        │   ├── 카테고리 선택 (선택)
        │   ├── 태그 선택/입력 (선택)
        │   ├── 저장 버튼
        │   └── 취소 버튼
        ├── ConfirmDialog (삭제 확인)
        │   ├── 경고 메시지 (소프트 삭제 / 영구 삭제 문구 분기)
        │   ├── 삭제 버튼 (destructive 스타일)
        │   └── 취소 버튼
        └── PriorityDropdown (우선순위 변경)
            ├── HIGH 항목 (빨강 아이콘)
            ├── MEDIUM 항목 (노랑 아이콘)
            └── LOW 항목 (파랑 아이콘)
```

---

## 2. 라우팅 구조

React Router v6 기반 라우팅 구조다. 인증이 필요한 경로는 `ProtectedRoute` 래퍼로 보호된다.

| 경로 | 컴포넌트 | 인증 필요 | 설명 |
|------|----------|-----------|------|
| `/` | — | 불필요 | `/login` 또는 `/todos`로 자동 리디렉트 |
| `/login` | `LoginPage` | 불필요 | 이메일·이름 입력 후 세션 발급. 이미 로그인된 경우 `/todos`로 리디렉트 |
| `/todos` | `TodosPage` | 필요 | 메인 할일 목록. 필터 바, TodoCard 목록, FAB 포함 |
| `/trash` | `TrashPage` | 필요 | 소프트 딜리트된 할일 목록. 복원 및 영구 삭제 가능 |
| `*` (catch-all) | `NotFoundPage` | 불필요 | 존재하지 않는 경로 접근 시 404 안내 페이지 |

### 리디렉트 규칙

| 상황 | 행동 |
|------|------|
| 비로그인 상태에서 `/todos` 접근 | `/login?redirect=/todos`로 리디렉트 |
| 비로그인 상태에서 `/trash` 접근 | `/login?redirect=/trash`로 리디렉트 |
| 로그인 상태에서 `/login` 접근 | `/todos`로 리디렉트 |
| 로그인 완료 후 | `redirect` 쿼리 파라미터가 있으면 해당 경로로, 없으면 `/todos`로 이동 |

---

## 3. 컴포넌트 계층 트리

React 컴포넌트의 전체 계층 구조다. 들여쓰기 깊이가 렌더 트리 깊이를 반영한다.

```
App
└── QueryClientProvider                  # React Query 전역 캐시
    └── BrowserRouter                    # React Router DOM
        ├── Toaster                      # react-hot-toast 전역 토스트
        └── Routes
            ├── /login → LoginPage
            │   └── LoginForm
            │       ├── EmailInput
            │       ├── NameInput
            │       └── SubmitButton
            └── ProtectedRoute           # 인증 미확인 시 /login 리디렉트
                ├── /todos → TodosPage
                │   ├── Header
                │   │   ├── Logo
                │   │   ├── NavLink ("할일 목록")
                │   │   ├── NavLink ("휴지통")
                │   │   ├── ShortcutHintBadge
                │   │   └── LogoutButton
                │   ├── FilterBar
                │   │   ├── StatusToggle ("TODO")
                │   │   ├── StatusToggle ("IN_PROGRESS")
                │   │   ├── StatusToggle ("DONE")
                │   │   ├── PriorityToggle ("HIGH")
                │   │   ├── PriorityToggle ("MEDIUM")
                │   │   ├── PriorityToggle ("LOW")
                │   │   ├── CategoryMultiSelect
                │   │   │   └── CategoryOption (×n)
                │   │   └── ResetFiltersButton
                │   ├── TodoList
                │   │   └── TodoCard (×n)
                │   │       ├── Checkbox
                │   │       ├── InlineEditTitle
                │   │       ├── PriorityIcon → PriorityDropdown (portal)
                │   │       │   ├── PriorityOption ("HIGH")
                │   │       │   ├── PriorityOption ("MEDIUM")
                │   │       │   └── PriorityOption ("LOW")
                │   │       ├── StatusBadge
                │   │       ├── DueDateDisplay
                │   │       ├── CategoryChip
                │   │       ├── TagChip (×n)
                │   │       └── DeleteButton → ConfirmDialog (portal)
                │   │           ├── DialogMessage
                │   │           ├── ConfirmDeleteButton
                │   │           └── CancelButton
                │   ├── EmptyState (conditional)
                │   │   ├── EmptyIllustration
                │   │   ├── EmptyMessage
                │   │   └── CreateFirstTodoButton (첫 사용 시만)
                │   └── TodoModal (conditional, portal)
                │       ├── TitleInput
                │       ├── DescriptionTextarea
                │       ├── DueDatePicker
                │       ├── PrioritySelect
                │       ├── CategorySelect
                │       ├── TagInput
                │       ├── SaveButton
                │       └── CancelButton
                └── /trash → TrashPage
                    ├── Header              # 공유 컴포넌트
                    ├── TrashNotice         # 30일 자동 삭제 안내
                    └── TrashList
                        └── TrashCard (×n)
                            ├── TitleDisplay
                            ├── DaysRemainingBadge
                            ├── RestoreButton
                            └── PermanentDeleteButton → ConfirmDialog (portal)
                                ├── DialogMessage
                                ├── ConfirmDeleteButton
                                └── CancelButton
```

### 공유 컴포넌트 목록

| 컴포넌트 | 사용처 | 비고 |
|----------|--------|------|
| `Header` | TodosPage, TrashPage | 네비게이션 포함 |
| `ConfirmDialog` | TodoCard, TrashCard | React Portal 사용 |
| `PriorityDropdown` | TodoCard | React Portal 사용 |
| `StatusBadge` | TodoCard | 상태별 색상 변형 |
| `DueDateDisplay` | TodoCard | D-day 계산 포함 |

---

## 4. 네비게이션 플로우

### 4-1. 인증 플로우

```
[사용자 접속]
      │
      ▼
  세션 확인
      │
  ┌───┴───┐
  │       │
없음      있음
  │       │
  ▼       ▼
/login  /todos
  │
  ▼
이메일 + 이름 입력
  │
  ▼
[시작하기] 클릭
  │
  ▼
POST /api/auth/login
  │
  ├── 성공 → 세션 저장 → /todos (또는 redirect 파라미터 경로)
  └── 실패 → 에러 토스트 표시, 폼 유지
```

### 4-2. 페이지 간 전환 플로우

```
/todos ◄──────────────────────────────────────► /trash
  │    헤더 NavLink "휴지통" 클릭                  │
  │    헤더 NavLink "할일 목록" 클릭               │
  │                                              │
  ▼                                              ▼
할일 목록 화면                              휴지통 화면
(필터 바 + 카드 목록)               (복원 / 영구 삭제 가능)
```

### 4-3. 모달 오픈/클로즈 플로우

```
/todos 화면
  │
  ├── [FAB / 새 할일 버튼] 클릭
  │         │
  │         ▼
  │    TodoModal 오픈 (오버레이)
  │         │
  │    ┌────┴────┐
  │    │         │
  │  [저장]    [취소]
  │    │         │
  │    ▼         ▼
  │  POST      모달 클로즈 (변경 없음)
  │  /api/todos
  │    │
  │    ├── 성공 → 모달 클로즈 → 목록 갱신 → 성공 토스트
  │    └── 실패 → 에러 토스트, 모달 유지
  │
  └── [삭제 버튼] 클릭 (TodoCard 내)
            │
            ▼
       ConfirmDialog 오픈
            │
       ┌────┴────┐
       │         │
     [삭제]    [취소]
       │         │
       ▼         ▼
  PATCH isDeleted=true  다이얼로그 클로즈
       │
       ├── 성공 → 카드 목록에서 제거 → 성공 토스트
       └── 실패 → 에러 토스트, 목록 원상 복구
```

### 4-4. 인라인 편집 플로우

```
TodoCard 제목 텍스트
  │
  └── 클릭 (또는 포커스)
            │
            ▼
     InlineEditTitle 활성화
     (contenteditable 또는 input 전환)
            │
     ┌──────┼──────┐
     │      │      │
  [Enter] [Blur] [Esc]
     │      │      │
     │      │      ▼
     │      │   변경 취소, 원본 텍스트 복원
     │      │
     └──────┘
            │
            ▼
       PATCH /api/todos/:id
            │
       ├── 성공 → 제목 업데이트
       └── 실패 → 원본 제목 복원 + 에러 토스트
```

### 4-5. 우선순위 변경 플로우

```
TodoCard 우선순위 아이콘
  │
  └── 클릭
        │
        ▼
  PriorityDropdown 오픈 (Portal)
        │
  ┌─────┼─────┐
  │     │     │
HIGH  MEDIUM  LOW
  │     │     │
  └─────┼─────┘
        │
        ▼
  PATCH /api/todos/:id { priority }
        │
  ├── 성공 → 아이콘 즉시 업데이트 (Optimistic UI)
  └── 실패 → 이전 우선순위 복원 + 에러 토스트
```

---

## 5. URL 설계 규칙

### 5-1. 현재 URL 목록

| URL | 메서드 (UI 관점) | 의미 |
|-----|-----------------|------|
| `/` | GET | 진입점. 인증 상태에 따라 리디렉트 |
| `/login` | GET | 로그인 페이지 |
| `/todos` | GET | 메인 할일 목록 |
| `/trash` | GET | 소프트 딜리트된 항목 목록 |

### 5-2. v2에서 추가될 URL

| URL | 의미 | 비고 |
|-----|------|------|
| `/todos/:id` | 할일 상세 페이지 | 설명, 댓글, 첨부파일 확장 시 |
| `/todos/:id/edit` | 할일 수정 전용 페이지 | 모달 방식을 페이지 방식으로 전환 시 |
| `/categories` | 카테고리 관리 페이지 | 카테고리 CRUD 전용 |
| `/settings` | 사용자 설정 페이지 | 알림, 테마, 계정 정보 |
| `/reports` | 통계/리포트 페이지 | 완료율, 기간별 추이 등 |

### 5-3. URL 파라미터 vs Query String 사용 기준

#### Path Parameter (`/todos/:id`) 사용 조건
- 리소스를 **식별**하는 값 (없으면 해당 리소스 자체가 존재하지 않음)
- 북마크·공유 가능한 영구적 식별자
- 예: `/todos/abc123` (특정 할일 상세)

#### Query String (`/todos?status=TODO&priority=HIGH`) 사용 조건
- **필터링**, **정렬**, **페이지네이션** 등 화면 상태를 URL에 직렬화하는 경우
- 값이 없어도 페이지 자체는 유효한 경우
- 예: `/todos?status=TODO,IN_PROGRESS&priority=HIGH&category=작업`
- 예: `/todos?page=2&sort=dueDate:asc`

#### 현재 버전의 필터 URL 직렬화 전략
현재 v1에서는 필터 상태를 URL Query String에 반영하지 않고 React 상태(useState)로만 관리한다. v2에서 URL 동기화를 도입할 때 아래 형식을 사용한다.

```
/todos?status=TODO,IN_PROGRESS&priority=HIGH,MEDIUM&category=개발,디자인
```

- 다중 값은 쉼표(`,`)로 구분
- 필터 초기화 시 해당 파라미터 제거 (빈 문자열 금지)

---

## 6. 상태별 화면 분기

### 6-1. /todos 페이지 상태 분기

| 화면 상태 | 표시 UI | 비고 |
|-----------|---------|------|
| **최초 진입 (데이터 로딩 중)** | 스켈레톤 카드 3개 | React Query `isLoading === true` |
| **필터 결과 없음** | EmptyState (필터 모드) — "조건에 맞는 할일이 없습니다" + 필터 초기화 버튼 | 데이터는 있으나 필터 결과가 0건 |
| **처음 사용 (할일 없음)** | EmptyState (첫 사용 모드) — "아직 할일이 없어요!" + "첫 할일 만들기" 버튼 | API 응답이 빈 배열, 필터 미적용 상태 |
| **데이터 정상 표시** | TodoCard 목록 + 필터 바 + FAB | — |
| **에러 발생** | 에러 배너 — "데이터를 불러오지 못했습니다" + 다시 시도 버튼 | React Query `isError === true` |
| **개별 카드 업데이트 중** | 해당 카드에 로딩 스피너 오버레이 (Optimistic UI 미적용 시) | Mutation `isPending` |

### 6-2. /trash 페이지 상태 분기

| 화면 상태 | 표시 UI | 비고 |
|-----------|---------|------|
| **로딩 중** | 스켈레톤 카드 3개 | — |
| **휴지통 비어있음** | EmptyState — "휴지통이 비어 있습니다" | isDeleted=true 항목 없음 |
| **데이터 정상 표시** | TrashCard 목록 + 안내 문구 | — |
| **에러 발생** | 에러 배너 + 다시 시도 버튼 | — |

### 6-3. EmptyState 분기 판단 로직

```
필터 적용 여부 확인
        │
   ┌────┴────┐
   │         │
필터 있음  필터 없음
   │         │
   ▼         ▼
"조건에    API 응답
 맞는 할일이  배열 길이
 없습니다"    │
+ 필터 초기화  ▼
   버튼     길이 === 0
            │
            ▼
         "아직 할일이
          없어요!"
        + 첫 할일 만들기
           버튼
```

### 6-4. 로딩 상태 스켈레톤 규격

| 요소 | 스켈레톤 표현 |
|------|--------------|
| 카드 제목 | 너비 60%, 높이 1rem, 회색 펄스 |
| 상태 뱃지 | 너비 4rem, 높이 1.25rem |
| 마감일 | 너비 5rem, 높이 1rem |
| 카드 전체 | 기본 카드 높이 유지, 내부만 스켈레톤 |

---

*이 문서는 앱의 구조가 변경될 때마다 업데이트되어야 한다. 라우팅 추가, 컴포넌트 리네이밍, 모달 구조 변경 시 반드시 반영할 것.*
