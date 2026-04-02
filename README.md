# replit-migration-skill

로컬 개발 코드를 Replit 배포 환경에 맞게 **검증하고 자동 수정**하는 [Claude Code](https://claude.ai/code) 스킬입니다.

## 기능

| 검증 항목 | 자동 수정 |
|-----------|:---------:|
| Prisma 마이그레이션 동기화 | 안내만 (로컬 DB 필요) |
| `.replit` NODE_ENV 설정 | O |
| `package.json` 스크립트 (start:replit, start:prod) | O |
| 환경변수 `.env.replit.example` 누락/포트 분리 | O |
| 테스트 계정 문서 (`TEST_ACCOUNTS.md`) 일치 여부 | O |
| 빌드 테스트 (`npm run build`) | 에러별 대응 |

## 설치

프로젝트 루트에서:

```bash
mkdir -p .claude/skills/replit-migrate
curl -o .claude/skills/replit-migrate/SKILL.md \
  https://raw.githubusercontent.com/Raphael-KR/replit-migration-skill/main/SKILL.md
```

## 사용법

Claude Code에서 슬래시 커맨드로 호출합니다.

```
# 검증만 (수정 없이 리포트만 출력)
/replit-migrate --check-only

# 검증 + 수정 (항목마다 사용자 확인)
/replit-migrate --fix

# 검증 + 전체 자동 수정 (확인 없이)
/replit-migrate --fix-all

# 기본값 (--fix 와 동일)
/replit-migrate
```

## 출력 예시

```
=== Replit 마이그레이션 결과 ===

[검증]
1. Prisma 마이그레이션:    OK
2. .replit 설정:           수정됨 (NODE_ENV 삭제)
3. package.json 스크립트:  수정됨 (start:prod에 NODE_ENV 추가)
4. 환경변수 (.env):        수정됨 (PORT 분리)
5. 테스트 계정 문서:       OK
6. 빌드 테스트:            OK

[수정 요약]
- 자동 수정: 3개 항목
- 수동 필요: 0개 항목

배포 준비 상태: READY
```

## 전제 조건

- [Claude Code](https://claude.ai/code) CLI 또는 IDE 확장이 설치되어 있어야 합니다
- 대상 프로젝트에 `.replit` 파일과 Replit 배포 구성이 있어야 합니다

## 프로젝트 구조

```
.
├── README.md
├── SKILL.md            # 메인 스킬 정의
└── SKILL-legacy.md     # 이전 pre-replit-deploy 스킬 (참고용)
```

## 라이선스

MIT
