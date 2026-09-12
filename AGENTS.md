# AGENTS.md — web-crawler (Codex / Claude Code dual-host)

이 레포는 URL과 수집 항목을 받아 사이트를 정찰·대량수집하고 엑셀로 내보내는 범용 웹 크롤링 에이전트다. **`CLAUDE.md`와 `.codex/skills/web-crawler/SKILL.md`가 *어떻게*에 대한 SSOT다.** 이 파일은 Codex용 **실행 계약**이다 — Claude Code는 Skill 런타임으로 같은 규율을 자동 적용받지만, Codex는 Skill 런타임이 없으므로 이 파일이 대신 강제한다.

## Git 리모트

- `origin` = 사용자 계정 fork(`emperorcastleai-sudo/web-crawler`, push 가능). `upstream` = 원본 저작자 repo(`byungjunjang/web-crawler`, push 불가·참고 및 pull 전용).
- **커밋·push는 항상 `origin`으로.** `upstream`에는 절대 push하지 않는다. fingerprints·output 등 사용자 수집 데이터가 원본 저작자 repo에 누적되지 않도록 한다.
- 원본 변경이 필요하면 `git fetch upstream` 후 필요한 범위만 병합하거나 체리픽한다. `git push upstream ...`은 시도하지 않는다.

## 추가 개발의 브랜치·PR 검증

새 기능, 동작·구조 변경, 여러 파일의 코드·테스트 수정만 Claude ↔ Codex 공용 GitHub PR 릴레이를 따른다. 절차 정본은 `/Users/hwangjaeseong/Claude/CLAUDE.md`의 “Claude ↔ Codex 릴레이”와 `docs/decisions/0001-git-relay.md`, `docs/decisions/0002-relay-conventions.md`, `docs/decisions/0006-pr-based-relay.md`다. 이 레포의 브랜치·PR·push 대상은 항상 `origin`이며 `upstream`에는 push하지 않는다.

문서 단독 정정, 생성 미러·도메인 목록 동기화, append-only 작업 로그는 추가 개발이 아니므로 브랜치·PR 의무 대상에서 제외한다. 다만 관련 검사와 검토는 수행한다.

## 최초 환경 셋업 (클론 직후 1회)

수집 전에 환경을 준비한다. **한 명령**으로 단계별 설치+검증을 하고, 이미 된 단계는 skip한다:

```powershell
# Windows (PowerShell) — 실행 정책 우회가 표준
powershell -ExecutionPolicy Bypass -File scripts\setup.ps1
```
```bash
# macOS / Linux
python -m venv .venv && . .venv/bin/activate && python scripts/bootstrap.py
```

단계: ① Python deps → ② 브라우저(Chromium) → ③ agent-browser(표준 정찰 도구) → ④ preflight 검증.
실패하면 "다음에 실행할 정확한 명령"이 출력된다. 모드: 기본 full(표준) / `--core-only`(agent-browser 제외) / `--skip-browser` / `-VerbosePip`(pip 상세 로그).

**`py` 런처 깨짐 자동 처리**: `setup.ps1`은 `py -3`/`python`/`python3`를 실제 실행해 3.10+를 확인하고 성공하는 쪽으로 venv를 만든다. `py -3`가 `No installed Python found!`로 실패하면 자동으로 `python`으로 fallback한다. 그래도 venv가 안 생기면 직접: `python -m venv .venv` → `.\.venv\Scripts\python.exe scripts\bootstrap.py`.

**수동/디버깅 시 실제 동작하는 명령 (Windows)**:
```powershell
python --version ; py -3 --version       # 어느 쪽이 동작하는지 먼저 확인 (py 깨졌으면 python 사용)
python -m venv .venv ; .\.venv\Scripts\Activate.ps1
pip install -r requirements.txt --progress-bar off    # 멈춘 듯하면 끝에 -v 추가
scrapling install                        # Chromium 1회 설치 (내부에서 playwright install chromium 수행 — 따로 또 X)
npm.cmd install -g agent-browser ; agent-browser.cmd install   # PowerShell은 .cmd 사용
python scripts\preflight.py              # 검증: core / agent-browser 분리 PASS·WARN·FAIL
```
- `python -m scrapling`은 동작 안 함 → `scrapling install`(venv 활성화) 또는 `.\.venv\Scripts\scrapling.exe install`.
- pip이 진행 없이 멈춘 듯하면 정상(대용량 휠 다운로드). 진행 확인: `.\.venv\Scripts\python.exe -m pip install -r requirements.txt --progress-bar off -v`.
- 검증은 `scripts/preflight.py`가 담당: **core(Python/Scrapling/Playwright)**와 **agent-browser**를 분리 보고. core 통과·agent-browser 실패면 "전체 설치 미완료"(종료코드 1). 전체 가이드는 `README.md` "처음 설치하기".

## 스킬 소스 (생성 미러)

- **`.claude/skills/`가 정본. `.codex/skills/`는 생성 미러**다 — 텍스트 안의 `.claude/skills` 경로만 `.codex/skills`로 치환된 것 외엔 byte-identical.
- **`.codex/skills/`를 직접 수정하지 말 것.** `.claude/skills/`를 고친 뒤 `python scripts/sync_codex_mirror.py`를 실행해 미러를 재생성한다. (어긋남 확인: `python scripts/sync_codex_mirror.py --check`)
- **문서의 "알려진 도메인" 목록도 생성물**이다 — `fingerprints/*/profile.json`이 SSOT. 새 프로필을 추가했으면 `python scripts/sync_domain_list.py`로 CLAUDE.md/README.md를 재생성한다. (어긋남 확인: `python scripts/sync_domain_list.py --check` / 테스트: `scripts/test_sync_domain_list.py`)

## 크롤링 요청을 받으면 — 필수 절차

사용자가 "크롤링/스크래핑/수집/~를 모아줘/입찰공고 수집" 등을 요청하면:

1. **즉흥 처리 금지.** `.codex/skills/web-crawler/SKILL.md`를 단계대로 실행한다. 절차를 요약하고 임의로 구현하지 않는다. **폴백 재구현 금지** — `requests`/`urllib`/`httpx`/`BeautifulSoup`로 직접 수집하거나 인라인으로 긁지 않는다. 수집은 항상 생성한 `crawl_script.py` 안의 **Scrapling 또는 Playwright**로만 한다.

2. **절대 규칙 0 — 도메인 히스토리 우선.** 정찰하기 전에 반드시 `fingerprints/<sanitized_domain>/profile.json`과 `output/<도메인>/`을 먼저 본다. 프로필이 있으면 `notes`/`fetcher_type`/`antibot_strategy`를 그대로 채택하고 정찰을 건너뛰어 Step 3으로 점프한다. profile.json이 있는데 무시하고 정찰부터 다시 하는 것은 금지(5~20분 비싼 작업 반복). 알려진 도메인 목록은 `CLAUDE.md` 의 생성 블록 참조.

3. **프로필 게이트.** Step 1-A(프로필 있으면 load) ↔ Step 5-A(수집 성공 직후 save/갱신, `notes` 필드 필수). Step 5-A를 빠뜨리면 수집 결과가 살아있어도 **"파이프라인 미완료"**로 보고한다.

## 정찰 도구 — Codex는 내장 브라우저(iab)가 표준

- Codex 정찰(Step 2)의 **표준·기본 도구는 내장 브라우저(iab)**다. `mcp__cua_repl`에서 `cua.getState()`로 iab를 확인한 뒤 임시 탭으로 구조·셀렉터·페이지네이션·건수를 확인한다. API 네트워크 감시나 로그인·쿠키 전달이 필요할 때만 `agent-browser`를 보조로 사용한다. Claude Code/Cowork의 기본은 계속 `agent-browser`다.
- **정찰 폴백 티어.** 실제 사용 티어를 profile.json `notes`에 남긴다.
  - **폴백 1 (Claude): Claude in Chrome** (`mcp__claude-in-chrome__*`) — **Claude 계열 host 전용**(Claude Code / Cowork). 사용자의 실제 Chrome을 조종해서 **실제 쿠키·실제 IP**가 그대로 붙는 게 최대 장점. Step 2 정찰 항목 5개는 전부 대체된다(검증 완료). 단 네트워크 감시에 제약이 있으니 반드시 SKILL.md Step 2 "Claude in Chrome 폴백" 절차를 따를 것.
  - **보조 (Codex): agent-browser** — iab의 일반 XHR/fetch 캡처 부재를 보완하거나 iab가 없는 세션에서 쓴다. 로그인·쿠키 전달도 전용 agent-browser 프로필에서만 처리한다.
  - **폴백 2 (공통): Scrapling `DynamicFetcher`** 또는 **Playwright `sync_api`**(`page.on("response")`로 XHR/API 캡처) — 양 host 공통, Python 의존성에 포함돼 항상 가능. (SKILL.md 규칙 1 예외와 동일.)
  - **host별 경로를 섞지 않는다.** Claude Code/Cowork는 `agent-browser → Claude in Chrome → 폴백 2`, Codex는 `iab → agent-browser(필요 시) → 폴백 2`다. Codex에서 Claude in Chrome을 찾지 않는다.
  어느 경우든 가능하면 `agent-browser.cmd install`로 표준 경로 복구를 먼저 시도한다.
- **수집은 폴백 대상이 아니다.** iab·agent-browser·Claude in Chrome은 **정찰 전용**이다 — 브라우저에서 전량 추출하는 것은 절대 규칙 2 위반. 수집은 어떤 host에서든 `crawl_script.py`(Scrapling/Playwright)로 한다.
- **원격 전용 환경(Cowork 등)에서 전 파이프라인 실행은 불가.** Cowork 샌드박스는 egress가 기본 "package managers only"(npm/PyPI/GitHub)라 대상 사이트 직접 접속이 막히고, 뚫어도 데이터센터 IP라 브라우저 세션이 필요한 도메인의 profile 레시피가 재현되지 않으며, VM에서 호스트 Chrome의 CDP 포트에 붙을 수 없어 `scripts/chrome_cdp.py` 경로가 통째로 죽는다. 원격에서는 **정찰만** 하고 profile.json을 갱신한 뒤, 수집은 로컬에서 실행한다.
- **수집·프로필·엑셀·CDP는 양 host 완전 동일**: 수집(Scrapling), 도메인 프로필(`scripts/domain_profile.py`), 엑셀(`scripts/export_excel.py`), 브라우저 세션이 필요한 사이트 대응(`scripts/chrome_cdp.py`), 진행 체크포인트(`scripts/progress.py`).

## 안전 — 하드룰 (위반 금지)

- **자동 접근 차단을 만나면 통지 후 사용자 선택** — CAPTCHA·WAF·봇 탐지는 법적으로 같은 보호조치다. 어느 쪽이든 **자동으로 넘어가지 않고 이음매를 통과할 때마다 한 번 알리고 사용자가 고른다**. '진행' 이면 그대로 간다 — 근거를 묻지도 검증하지도 않는다. 통지를 면제하는 것은 도메인이 아니라 그 프로필이 **지금 들고 있는** `consent` 기록이다(sticky) — 사다리 A 로 내려가 프로필이 배포 대상이 되면 그 기록은 지워지므로(사용자의 통지 이력을 배포되는 파일에 실어 보내지 않는다), 사이트가 나중에 새로 막으면 다시 통지한다. 상세는 `.codex/skills/web-crawler/SKILL.md` Step 3 "이음매 통지 게이트".
- **CAPTCHA 자동 풀이 금지** — 위 통지 게이트와 별개다. reCAPTCHA/hCaptcha 를 **프로그램으로 푸는 것**은 하지 않는다. 사용자가 agent-browser 로 직접 풀고 이어가는 것은 가능하다.
- **로그인 자격증명 저장 금지** — ID/PW를 코드·메모리·파일에 저장하지 않는다. 사용자가 직접 로그인 → 쿠키만 추출(`output/<도메인>/cookies.json`, `.gitignore`가 차단).
- **robots.txt 제한** 발견 시(`Disallow: /` 또는 대상 경로 차단) 진행 여부를 사용자에게 묻는다.
- **PII 감지 필수** — 수집 데이터에 전화번호/주민번호/이메일 등이 섞이면 `detect_pii(data)`로 경고하고 보고한다.
- **법적 위험이 큰 요청은 구체적으로 경고** — 저작권 침해 목적의 본문 복제(분량 축)·개인정보 대량 수집(성격 축)·명시적으로 금지된 재배포(목적 축). **어느 축이 왜 걸리는지 짚어서 알린 뒤 진행 여부는 사용자가 정한다** — 근거를 묻지도 검증하지도 않는다. **약관이 크롤링을 금지한다는 사실만으로는 여기 해당하지 않는다** — 그건 접근의 계약 층이고, 위 통지 게이트로 간다.
- **수집 0건이면 즉시 중단·보고** — 계속 시도하면 ban 위험.

> **위 '경고' 규칙과 통지 게이트는 같은 층위다.** 경고는 요청이 **어떤 위험 축에 걸리는지**를,
> 게이트는 **기술적 차단을 만났다는 사실**을 알린다. 알리는 대상만 다를 뿐 둘 다 알리는 데서
> 끝나고 고르는 쪽은 사용자다 — 어느 쪽도 요청 자체를 막지 않으며, 근거를 묻지도 검증하지도
> 않는다.
>
> 이 문서가 정의하는 것은 **도구의 동작**이다. 실행하는 AI 에이전트 자신의 판단 기준은 별개로
> 작동하며 이 문서가 그것을 대신 약속하지 않는다 — `ACCEPTABLE_USE.md` 참조.

## 빠른 참조

| 무엇 | 경로 |
|------|------|
| 워크플로우 7단계 | `.codex/skills/web-crawler/SKILL.md` |
| Fetcher 코드 템플릿 | `.codex/skills/web-crawler/references/fetcher-patterns.md` |
| 안티봇(Akamai/Cloudflare/SPA 세션) | `.codex/skills/web-crawler/references/antibot-strategies.md` |
| 수집 실패 진단 | `.codex/skills/web-crawler/references/troubleshooting.md` |
| 프로젝트 규칙·도구 분리 SSOT | `CLAUDE.md` |
