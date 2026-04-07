# Operations Policy
## 사내 작업 관리 도구 — Todo List

> 배포, 환경 관리, DB 유지보수, 보안, 장애 대응에 관한 운영 규칙.

---

## 1. 환경 구성

### 현재 환경

| 환경 | 용도 | DB | 비고 |
|------|------|-----|------|
| local | 개발자 로컬 | SQLite (`dev.db`) | 설치 불필요 |
| staging | 통합 테스트 | PostgreSQL 15 | 운영과 동일 인프라 |
| production | 사내 서비스 | PostgreSQL 15 | |

### 환경별 설정 원칙

- `NODE_ENV=development` — SQLite, 쿼리 로깅 활성화 (`prisma.ts`의 `log` 옵션)
- `NODE_ENV=production` — PostgreSQL, 쿼리 로깅 비활성화, 에러 상세 숨김
- `CORS_ORIGIN` — 운영에서는 반드시 실제 프론트엔드 도메인으로 제한 (`*` 금지)

---

## 2. DB 관리

### 개발 → 운영 스키마 마이그레이션

현재 `backend/prisma/schema.prisma`는 SQLite 기준.
**운영(PostgreSQL) 전환 시 필수 작업**:

```prisma
// 1. datasource 변경
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// 2. title 필드 길이 제한 복원
title String @db.VarChar(200)

// 3. enum 복원 (String → 실제 enum)
enum Priority    { HIGH MEDIUM LOW }
enum TodoStatus  { TODO IN_PROGRESS DONE DELETED }
```

그 후:
```bash
# 마이그레이션 파일 생성 (운영 배포 전 반드시 staging에서 먼저 검증)
npx prisma migrate dev --name init

# 운영 적용
npx prisma migrate deploy
```

### 마이그레이션 규칙

- 컬럼 추가: 항상 `DEFAULT` 또는 nullable로 추가 (기존 데이터 영향 없음)
- 컬럼 삭제: 코드에서 먼저 제거 → 배포 → 다음 배포에서 스키마 삭제 (2단계)
- 컬럼 이름 변경: 직접 rename 금지 → 신규 컬럼 추가 + 데이터 마이그레이션 + 구 컬럼 삭제
- 운영 DB에 `prisma db push` 사용 금지 (마이그레이션 히스토리 손실)

### 정기 유지보수

| 작업 | 주기 | 방법 |
|------|------|------|
| 소프트 딜리트 30일 영구 삭제 | 매일 | `purgeWorker.ts` 구현 필요 (v2) |
| DB 백업 | 매일 | pg_dump 또는 클라우드 자동 백업 |
| 미발송 리마인더 정리 | 주간 | 마감 경과 + 미발송 레코드 `isSent=true` 처리 |

### 현재 미구현: 30일 자동 영구 삭제

`purgeWorker.ts` 구현 전까지 수동 실행:

```sql
-- 30일 경과 소프트 딜리트 할일 영구 삭제
DELETE FROM todos
WHERE is_deleted = true
  AND deleted_at < datetime('now', '-30 days');  -- SQLite
  -- AND deleted_at < now() - interval '30 days';  -- PostgreSQL
```

---

## 3. 보안 정책

### JWT 토큰

| 항목 | 정책 |
|------|------|
| Access Token 만료 | 15분 (`JWT_EXPIRES_IN`) |
| Refresh Token 만료 | 7일 (`JWT_REFRESH_EXPIRES_IN`) |
| 시크릿 키 길이 | 최소 64자 랜덤 문자열 |
| 저장 위치 | 프론트엔드 `localStorage` (현재) → 운영에서 `httpOnly cookie` 전환 권장 |
| 토큰 갱신 | Access Token 만료 시 Refresh Token으로 자동 갱신 |

**운영 전 필수**: `localStorage` → `httpOnly cookie` 전환
현재 `frontend/src/api/client.ts`의 토큰 처리를 쿠키 기반으로 변경해야 XSS 공격으로부터 토큰 보호 가능.

### HTTPS

- 운영 환경에서 HTTPS 필수
- HTTP 접근 시 HTTPS로 리디렉트 (Nginx/로드밸런서 레벨 설정)

### 데이터 격리

- 모든 보호 라우트에 `requireAuth` 미들웨어 적용 (현재 모든 라우트에 적용됨)
- DB 쿼리에 `userId` 조건 필수 (현재 모든 쿼리에 포함됨)
- 타인 리소스 접근 시 `403` 반환 (리소스 존재 여부 노출 금지 고려 — 현재 `404` 반환)

### 입력 유효성 검증

- 모든 API 입력은 Zod 스키마로 검증 (현재 모든 라우트에 적용)
- 제목 최대 길이 서버 측 강제: `z.string().min(1).max(200)`
- SQL Injection: Prisma ORM이 파라미터화 쿼리 처리 → 직접 raw query 사용 시 반드시 파라미터 바인딩

---

## 4. 배포

### 현재 (로컬 실행)

```bash
# 백엔드
cd backend && npm run dev        # 개발 (tsx watch)
cd backend && npm run build && npm run start  # 프로덕션 빌드

# 프론트엔드
cd frontend && npm run dev       # 개발 (Vite HMR)
cd frontend && npm run build     # 프로덕션 빌드 → dist/
```

### 권장 운영 배포 구성

```
[인터넷]
    │
    ▼
[Nginx / Reverse Proxy]
    ├── / → frontend/dist/ (정적 파일 서빙)
    └── /api → http://localhost:4000 (백엔드 프록시)
```

**프론트엔드**: `dist/` 디렉토리를 Nginx 또는 CDN(S3 + CloudFront)으로 서빙
**백엔드**: PM2 또는 Docker 컨테이너로 실행

### PM2 예시 (백엔드)

```bash
npm install -g pm2
cd backend
npm run build
pm2 start dist/index.js --name "todo-backend" --env production
pm2 save
pm2 startup
```

### 환경변수 운영 체크리스트

- [ ] `DATABASE_URL` PostgreSQL 연결 문자열
- [ ] `JWT_SECRET` 64자 이상 랜덤 문자열
- [ ] `JWT_REFRESH_SECRET` 64자 이상 랜덤 문자열 (JWT_SECRET과 다르게)
- [ ] `CORS_ORIGIN` 프론트엔드 실제 도메인
- [ ] `NODE_ENV=production`
- [ ] HTTPS 인증서 적용 확인

---

## 5. 모니터링 & 로깅

### 현재 로깅

- 백엔드: `console.error`로 에러 출력 (라우트별 prefix 포함: `[Todos]`, `[Auth]` 등)
- 리마인더 워커: 발송 시 `console.log` 출력
- Prisma: `NODE_ENV=development`에서 쿼리 로그 출력

### 운영 권장 로깅

- `console.log/error` → `winston` 또는 `pino` 구조화 로거로 교체
- 로그 레벨: `error` / `warn` / `info` / `debug`
- 에러 추적: Sentry 연동 권장

### 핵심 모니터링 지표

| 지표 | 목표 | 측정 위치 |
|------|------|-----------|
| API 응답 시간 p95 | < 300ms | 애플리케이션 로그 또는 APM |
| 에러율 (5xx) | < 0.1% | 애플리케이션 로그 |
| 가용성 | 99.5% (월간) | 외부 헬스체크 (`GET /health`) |
| 리마인더 발송 성공률 | > 99% | 리마인더 워커 로그 |

### 헬스체크 엔드포인트

```
GET /health
→ 200 { "status": "ok", "timestamp": "..." }
```

로드밸런서 및 외부 모니터링 도구(UptimeRobot 등)에서 이 엔드포인트를 주기적으로 호출한다.

---

## 6. 장애 대응

### 백엔드 응답 없음

1. `GET /health` 확인 → 500/timeout이면 프로세스 재시작
2. DB 연결 확인: `DATABASE_URL` 환경변수 및 DB 서버 상태
3. 로그 확인: `pm2 logs todo-backend`

### DB 연결 오류

1. `DATABASE_URL` 확인
2. PostgreSQL 서버 상태 확인
3. 최대 연결 수 초과 시 `connection_limit` 조정 (`DATABASE_URL?connection_limit=5`)

### 리마인더 미발송

1. `reminderWorker.ts` 로그 확인
2. `reminders` 테이블에서 `is_sent=false AND remind_at < now()` 조회
3. 수동 발송 후 `is_sent=true` 업데이트
4. (v2) 실 이메일 발송 구현 후 SMTP 연결 확인

### JWT 시크릿 변경 필요 시

1. 새 시크릿으로 환경변수 업데이트
2. 백엔드 재시작
3. **기존 모든 토큰 무효화** (사용자 재로그인 필요)
4. 사용자에게 재로그인 안내 공지
