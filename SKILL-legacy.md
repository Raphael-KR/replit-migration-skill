---
name: pre-replit-deploy
description: "[Deprecated] /replit-migrate 스킬로 대체됨. 검증만 필요하면 /replit-migrate --check-only 사용"
argument-hint: ""
disable-model-invocation: true
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash
---

# [Deprecated] Replit 배포 전 로컬 검증 체크리스트

> **이 스킬은 `/replit-migrate` 스킬로 대체되었습니다.**
> - 검증만: `/replit-migrate --check-only`
> - 검증 + 수정: `/replit-migrate --fix`
> - 전체 자동: `/replit-migrate --fix-all`

아래는 레거시 체크리스트입니다 (참고용).

사용자가 로컬 코드를 Replit에 배포하기 전에, 아래 항목을 **순서대로** 검증하고 결과를 리포트하라.
문제가 발견되면 즉시 사용자에게 알리고, 수정 여부를 확인받은 뒤 진행하라.

## 1. Prisma 마이그레이션 파일 동기화 확인

- `apps/backend/prisma/schema.prisma`의 모든 모델/필드가 `apps/backend/prisma/migrations/` 내의 `.sql` 파일에 반영되어 있는지 확인
- 방법: `npx prisma migrate diff --from-migrations apps/backend/prisma/migrations --to-schema-datamodel apps/backend/prisma/schema.prisma` 실행
- diff 결과가 비어있지 않으면 **마이그레이션 파일 누락**이다
- 누락 발견 시 사용자에게 알리고, `npx prisma migrate dev --name <설명>`을 로컬에서 실행하도록 안내 (이 명령은 로컬 DB 연결이 필요하므로 사용자가 직접 실행해야 함)

## 2. 환경변수 문서 정합성 확인

- `.env.replit.example` 파일이 존재하는지 확인
- 백엔드 코드(`apps/backend/src/`)에서 `process.env.` 또는 `configService.get`으로 참조하는 환경변수 목록을 수집
- `.env.replit.example`에 해당 변수들이 문서화되어 있는지 대조
- 누락된 변수가 있으면 리포트
- 특히 다음 값이 올바른지 확인:
  - `PORT=3000` (프론트엔드용)
  - `BACKEND_PORT=3001` (백엔드용)
  - `NEXT_PUBLIC_API_URL` 값이 Replit 환경에 적합한지

## 3. devDependencies 빌드 의존성 확인

- root `package.json`의 `devDependencies`에 빌드 필수 패키지(`turbo`, `typescript`, `concurrently`)가 포함되어 있는지 확인
- `.replit` 파일의 `[env]` 섹션에 `NODE_ENV=production`이 있으면 경고 (npm install 시 devDependencies가 설치 안 됨)

## 4. 빌드 테스트

- `npm run build`를 실행하여 에러 없이 빌드되는지 확인
- 빌드 실패 시 에러 내용을 리포트

## 5. seed 데이터와 테스트 계정 문서 일치 확인

- `apps/backend/prisma/seed.ts`의 테스트 계정 정보(이메일, 비밀번호, 역할)를 읽음
- `docs/TEST_ACCOUNTS.md`의 내용과 대조
- 불일치 항목이 있으면 리포트

## 6. start:replit 스크립트 확인

- root `package.json`의 `start:replit` 스크립트가 존재하고, 다음 순서를 포함하는지 확인:
  1. `db:generate` (Prisma Client 생성)
  2. `db:migrate:prod` (마이그레이션 적용)
  3. `build` (전체 빌드)
  4. `start:prod` (서버 시작)

## 최종 리포트

모든 검증이 끝나면 아래 형식으로 결과를 요약하라:

```
[Replit 배포 전 검증 결과]
1. Prisma 마이그레이션: OK / 누락 있음 (상세)
2. 환경변수 문서: OK / 누락 있음 (상세)
3. devDependencies: OK / 경고 (상세)
4. 빌드 테스트: OK / 실패 (상세)
5. 테스트 계정 문서: OK / 불일치 (상세)
6. start:replit 스크립트: OK / 문제 있음 (상세)

배포 준비 상태: READY / NOT READY
```
