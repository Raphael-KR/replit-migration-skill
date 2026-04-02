---
name: replit-migrate
description: 로컬 코드를 Replit 배포 환경에 맞게 검증하고 자동 수정하는 마이그레이션 스킬
argument-hint: "[--check-only | --fix | --fix-all]"
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Edit, Write, Grep, Glob, Bash, Agent
---

# Local → Replit 마이그레이션 스킬

로컬 개발 코드를 Replit 배포 환경에 맞게 **검증 + 자동 수정**하는 통합 스킬이다.

## 실행 모드

인자에 따라 동작이 달라진다:

| 인자 | 동작 |
|------|------|
| `--check-only` | 검증만 수행, 수정하지 않음 (리포트만 출력) |
| `--fix` | 검증 후 문제 발견 시 **각 항목마다 사용자 확인** 받고 수정 |
| `--fix-all` | 검증 후 문제 발견 시 **전부 자동 수정** (확인 없이) |
| (인자 없음) | `--fix` 와 동일하게 동작 |

---

## Phase 1: 검증 (Validation)

아래 항목을 **순서대로** 검증하고 각 항목의 상태를 기록한다.

### 1.1 Prisma 마이그레이션 동기화

- `apps/backend/prisma/schema.prisma`와 `apps/backend/prisma/migrations/` 내 `.sql` 파일 비교
- 방법: `cd apps/backend && npx prisma migrate diff --from-migrations prisma/migrations --to-schema-datamodel prisma/schema.prisma` 실행
- diff 결과가 비어있지 않으면 **마이그레이션 누락**
- **수정 불가 항목**: 마이그레이션 생성은 로컬 DB 연결이 필요하므로, 누락 발견 시 사용자에게 `npx prisma migrate dev --name <설명>` 직접 실행을 안내한다

### 1.2 `.replit` 파일 검증

- `.replit` 파일이 존재하는지 확인
- `[env]` 섹션에 `NODE_ENV = "production"`이 있는지 확인
  - 있으면 **문제**: `npm install` 시 devDependencies가 설치되지 않아 빌드 실패
  - **수정**: `[env]` 섹션과 `NODE_ENV = "production"` 줄을 삭제

### 1.3 `package.json` 스크립트 검증

- root `package.json` 확인:
  - `start:replit` 스크립트가 존재하고, 아래 순서를 포함하는지:
    1. `db:generate`
    2. `db:migrate:prod`
    3. `build`
    4. `start:prod`
  - `start:prod` 스크립트에 `NODE_ENV=production`이 접두어로 포함되어 있는지
    - 없으면 **문제**: `.replit`에서 NODE_ENV를 제거한 경우 백엔드가 production 모드를 인식 못함
    - **수정**: `start:prod` 값 앞에 `NODE_ENV=production` 추가
  - `devDependencies`에 빌드 필수 패키지(`turbo`, `typescript`, `concurrently`)가 있는지

### 1.4 환경변수 정합성

- `.env.replit.example` 파일 존재 확인
- 백엔드 코드(`apps/backend/src/`)에서 `process.env.` 또는 `configService.get`으로 참조하는 환경변수 수집
- 프론트엔드 코드(`apps/frontend/src/`)에서 `NEXT_PUBLIC_` 접두어 환경변수 수집
- `.env.replit.example`에 누락된 변수가 있으면 리포트
- 포트 설정 검증:
  - `PORT=3000` (프론트엔드) / `BACKEND_PORT=3001` (백엔드) 로 분리되어야 함
  - 둘 다 `PORT=3001`만 있으면 **문제**: Next.js와 NestJS가 같은 포트를 참조해 충돌
  - **수정**: `PORT=3000`, `BACKEND_PORT=3001`로 분리
- `NEXT_PUBLIC_API_URL` 값 검증:
  - `/api/v1`이면 **문제**: Next.js rewrites 경유 시 `/api/v1/api/v1/...` 경로 중복 가능
  - **수정**: `http://localhost:3001`로 변경 (또는 Replit 환경에서는 상대경로 `/api/v1` 유지하되 Next.js rewrites 설정 확인)

### 1.5 시드 데이터 / 테스트 계정 문서 일치

- `apps/backend/prisma/seed.ts`의 테스트 계정 정보(이메일, 비밀번호, 역할) 읽기
- `docs/TEST_ACCOUNTS.md` 내용과 대조
- 불일치 시 **수정**: `docs/TEST_ACCOUNTS.md`를 seed.ts 기준으로 업데이트

### 1.6 빌드 테스트

- `npm run build` 실행하여 에러 없이 빌드되는지 확인
- 실패 시 에러 내용을 리포트
- 빌드 에러가 Replit 환경 특화 문제(Prisma Bytes 타입, 경로 문제 등)인 경우 자동 수정 시도

---

## Phase 2: 수정 (Fix)

`--check-only`가 아닌 경우, Phase 1에서 발견된 문제를 수정한다.

### 수정 우선순위

1. `.replit` NODE_ENV 제거 (다른 수정의 전제 조건)
2. `package.json` 스크립트 수정
3. `.env.replit.example` 포트/URL 수정
4. `docs/TEST_ACCOUNTS.md` 최신화
5. 빌드 에러 수정

### 수정 원칙

- 각 수정은 개별적으로 수행하고 결과를 확인
- `--fix` 모드에서는 각 수정 전에 변경 내용을 사용자에게 보여주고 승인 요청
- `--fix-all` 모드에서는 전부 자동 적용
- Prisma 마이그레이션 누락은 자동 수정 불가 (로컬 DB 필요) — 안내만 출력

---

## Phase 3: 최종 리포트

모든 작업이 끝나면 아래 형식으로 결과를 요약한다:

```
=== Replit 마이그레이션 결과 ===

[검증]
1. Prisma 마이그레이션:    OK / 누락 있음 (상세)
2. .replit 설정:           OK / 수정됨 / 수정 필요
3. package.json 스크립트:  OK / 수정됨 / 수정 필요
4. 환경변수 (.env):        OK / 수정됨 / 누락 있음
5. 테스트 계정 문서:       OK / 수정됨 / 불일치
6. 빌드 테스트:            OK / 실패 (상세)

[수정 요약]
- 자동 수정: N개 항목
- 수동 필요: N개 항목 (상세 안내 포함)

배포 준비 상태: READY / NOT READY
```

`NOT READY`인 경우, 남은 수동 작업을 단계별로 안내한다.

---

## 참고 문서

이 스킬의 검증/수정 기준은 아래 문서를 근거로 한다:
- `docs/REPLIT_DEPLOYMENT.md` — Replit 배포 가이드
- `docs/REPLIT_POST_DEPLOY_TASKS.md` — 배포 후속 수정 작업 목록
- `.replit` — Replit 실행 설정
- `.env.replit.example` — 환경변수 템플릿
