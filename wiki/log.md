# Work Log
> append-only — 기존 항목 수정·삭제 금지. 작업 완료마다 한 줄 추가.

형식: `## [YYYY-MM-DD] {작업유형} | {제목}`

## [2026-09-13] decision | Claude↔Codex 릴레이 방식 분리 확정 — `fingerprints/profile.json`(수집 레시피) 동기화는 PR 불필요, 그냥 git commit+push/pull(이미 이 레포 설계상 commit 대상). PR 릴레이(ADR 0006)는 크롤러 도구 자체 코드(scripts/*.py) 변경 시에만 기존 Dev_Projects 흐름 그대로 적용. 크롤링 진행 상황 공유는 PR 대신 각자 `wiki/log.md`에 "어느 도메인을 누가 언제 수집했는지" 한 줄 기록으로 대체.
## [2026-09-13] setup | 산출물 공유 구조 변경 — 로컬 `output/`을 `Work_Brain/crawler/output/`로 심볼릭 링크. Claude/Codex 어느 쪽이 수집을 실행하든(토큰 비용 고려한 라우팅) 결과물(xlsx 등)이 한 곳에 모이게 함. `output/`은 원래 gitignore 대상이라 레포 구조·git 이력엔 영향 없음. `fingerprints/`(도메인 레시피)는 이미 git commit 대상이라 별도 조치 불필요 — push/pull로 자연히 동기화. Codex 쪽 핸드오프 문서(`~/CODEX/shared-web-crawler-SETUP.md`)에 동일 심볼릭 링크 절차 추가.
## [2026-09-13] setup | 초기 설치 — github.com/byungjunjang/web-crawler 클론(master) + venv + `scripts/bootstrap.py` 실행(Python 패키지·Chromium·agent-browser). preflight 전체 통과(CORE 13/13, agent-browser 3/3). books.toscrape.com 대상 실제 수집 5건 + 엑셀 저장 실행 검증 완료. 이 프로젝트는 Claude/Codex 저자·검증자 역할분리 대상이 아닌 공용 도구(레포 자체가 양 host SSOT로 설계됨) — 폴더명에 `shared-` 접두어로 구분. 정찰 브라우저는 레포 기본값 agent-browser 채택.
## [2026-09-13] docs | 프로젝트 `CLAUDE.md` 컨텍스트 예산 초과(412줄 → 예산 110줄) 정리 — SKILL.md/references에 이미 있는 절차 상세(사다리 흐름도, fetcher 코드 패턴, 쿠키 주입, profile save 코드)는 중복 제거하고 포인터만 남겨 84줄로 축소. 누락 위험 있던 에러별 대응표·Rate Limiting 기준표는 `references/troubleshooting.md`로 이관해 보존. 커밋: 375a389.
## [2026-09-13] fix | git 리모트가 사용자 소유가 아닌 원저작자(byungjunjang) repo였음을 발견(push 권한도 없었음) — `byungjunjang/web-crawler`를 `emperorcastleai-sudo/web-crawler`로 fork 후 `origin`을 fork로, 원본은 `upstream`으로 재배선. 이후 fingerprints/profile.json 등 사용자 수집 데이터가 원저작자 repo로 쌓이는 걸 방지. `AGENTS.md`에 "커밋·push는 항상 origin, upstream엔 절대 push 금지" 하드룰 추가 — 커밋: 7b1cb96.
