# 범용 웹 크롤링 에이전트

URL과 수집 항목을 자연어로 설명받으면 사이트를 정찰하고 데이터를 대량 수집해 엑셀로 정리하는 에이전트.

절차 전문은 `.claude/skills/web-crawler/SKILL.md`(Step 1~6, Step 1-A/5-A 게이트 포함)를 따른다. 이 파일은 그 스킬을 트리거하는 프로젝트 규칙만 담는다.

## 최초 환경 셋업 (클론 직후 1회)

```powershell
powershell -ExecutionPolicy Bypass -File scripts\setup.ps1     # Windows (venv 자동 생성)
```
```bash
python -m venv .venv && . .venv/bin/activate && python scripts/bootstrap.py   # macOS/Linux
```

- 검증만: `python scripts/preflight.py` (core / agent-browser 분리 PASS·WARN·FAIL, 설치는 안 함)
- `python -m scrapling`은 동작 안 함 → `scrapling install`
- 비개발자용 포함 전체 가이드: `README.md`의 "처음 설치하기"

## ★ 절대 규칙 0: 도메인 히스토리 우선

새 수집 요청을 받으면 **정찰 전에 반드시** 먼저 본다:

1. `fingerprints/<sanitized_domain>/profile.json` — 검증된 수집 레시피 (fetcher_type/antibot_strategy/selectors/notes)
2. `output/<도메인>/` — 이전 실행 폴더 (`crawl_script.py`가 profile에 없는 미세 디테일의 보조 reference)

프로필이 있으면 notes부터 읽고 fetcher/selector를 그대로 채택한다. 정찰부터 다시 하지 않는다 — 5~20분의 비싼 작업을 반복하는 행위다. 상세 분기(사다리 B 재통지 조건, consent sticky 규칙)는 SKILL.md Step 1-A 참조.

<!-- BEGIN GENERATED: domain-list -->
<!-- 이 블록은 scripts/sync_domain_list.py 가 생성한다. 직접 수정하지 말 것. -->

### 알려진 도메인 (14개 profile commit됨)

`books.toscrape.com`, `builtini.co.kr`, `celimax.co.kr`, `data.seoul.go.kr`, `db.itkc.or.kr`, `g2b.go.kr`, `guesskorea.com`, `made-in-china.com`, `wanted.co.kr`, `www.11st.co.kr`, `www.fss.or.kr`, `www.gsmarena.com`, `www.k-startup.go.kr`, `www.kurly.com` — 정찰 없이 바로 수집 시도 가능.

<!-- END GENERATED: domain-list -->

새 도메인 프로필 commit 후: `python scripts/sync_domain_list.py` (목록 재생성, 손으로 안 고침)

## 범위 / 운영 안전 규칙

**포함**: 사이트 정찰, 로그인 대응, 동적 콘텐츠, pagination, 대량 수집, 엑셀 출력.

- **자동 접근 차단(CAPTCHA·WAF·봇탐지) 만나면 통지 후 사용자 선택** — 심사 아님, '진행'이면 근거 안 물음. 상세: SKILL.md Step 3 "이음매 통지 게이트"
- **CAPTCHA 자동 풀이 금지** (사용자가 agent-browser로 직접 푸는 건 가능)
- **로그인 자격증명 자동 저장 금지** — 사용자가 직접 로그인 → 쿠키만 추출
- **법적 위험 요청(저작권 본문 복제/PII 대량 수집/금지된 재배포)은 축을 짚어 경고 후 사용자 선택** — 약관상 금지만으로는 해당 안 함(그건 통지 게이트로)
- **robots.txt 제한 시 사용자 확인**, **PII 감지 시 `detect_pii(data)`로 경고**

에이전트 자신의 판단 기준과 도구 동작 규정의 관계는 `ACCEPTABLE_USE.md` 참조.

## 도구 역할 분리

| 작업 | 도구 |
|------|------|
| 도메인 히스토리 조회/저장 | `scripts/domain_profile.py` (`DomainProfile`) |
| 정찰 (표준) | agent-browser |
| 정찰 폴백 | Claude in Chrome / ChatGPT Chrome Browser Use — agent-browser 불가 시만 |
| 대량 데이터 수집 | Scrapling — **agent-browser로 대량 수집 절대 금지** |
| 통지 이후 브라우저 세션 필요 시 | Chrome CDP (`scripts/chrome_cdp.py`) |
| 진행상황 체크포인트 | `scripts/progress.py` |
| 엑셀 출력 | `scripts/export_excel.py` |

**원격 전용 환경(Cowork 등)은 정찰까지만** — egress 제한·데이터센터 IP·호스트 CDP 미접속으로 수집 재현 불가. 수집은 로컬에서.

## 검증 통과 기준 (Step 5)

수집 건수 목표 대비 90%↑, 전체 데이터 95%↑ 유효, 필드별 null 비율 10%↓. 미달 시 최대 2회 재시도, 재실패해도 수집분으로 진행하되 사용자에게 경고.

## 스킬/레퍼런스

- `.claude/skills/web-crawler/SKILL.md` — 7단계 전체 흐름
- `references/fetcher-patterns.md` — Fetcher별 코드 템플릿, 사다리 A/B, Rate Limiting, 쿠키 전달, Spider/Infinite Scroll 기준
- `references/antibot-strategies.md` — Akamai/Cloudflare/SPA 세션 대응
- `references/troubleshooting.md` — 수집 실패 진단, 에러별 대응표

## 저장 디렉터리

```
output/<도메인>/<주제_YYYYMMDD_HHMMSS>/   # crawl_result.xlsx, raw_data.json, progress.json, crawl_script.py (gitignore)
fingerprints/<sanitized_domain>/profile.json  # 배포 판정 통과분만 commit (default-deny + whitelist)
```

`fingerprints/**`는 default-deny, whitelist는 `scripts/sync_domain_list.py`가 생성(손으로 안 고침). `cookies*/`, `*auth*`, `*token*`, `*secret*`은 whitelist 뒤에서 재차단. profile.json에 토큰/쿠키 절대 금지 — commit 전 `git diff --cached fingerprints/`로 확인.
