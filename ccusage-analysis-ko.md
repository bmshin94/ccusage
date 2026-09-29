# ccusage 전수조사 분석 정리 (한국어)

> 작성일: 2026-09-29
> 대상 저장소: <https://github.com/bmshin94/ccusage> (포크)
> 원본 저장소: <https://github.com/ccusage/ccusage>
> 공식 문서: <https://ccusage.com>
> npm 패키지: <https://www.npmjs.com/package/ccusage>
> 라이선스: MIT (원작자 [@ryoppippi](https://github.com/ryoppippi))

---

## 1. 이게 뭐하는 프로젝트인가

`ccusage`는 **로컬에 쌓인 AI 코딩 에이전트의 로그 파일을 읽어서 토큰 사용량과 비용(USD)을
집계해 터미널 표로 보여주는 CLI 도구**다.

### 동작 원리

```
① 에이전트가 대화마다 로그를 남김 (예: ~/.claude/projects/**/*.jsonl)
       ↓
② ccusage가 JSONL을 병렬 파싱 → input / output / cache 토큰 집계
       ↓
③ LiteLLM · models.dev 가격표와 모델명을 매칭해 달러 환산
       ↓
④ 터미널 표 또는 --json 출력
```

핵심은 **100% 로컬 파일 읽기**라는 점이다. 로그인·API 키·서버 전송이 전혀 없다.

### 지원 소스 (18종)

Claude Code, Codex, Gemini CLI, GitHub Copilot CLI, OpenCode, Amp, Droid, Codebuff,
Hermes Agent, pi-agent, Goose, OpenClaw, Kilo, Kimi, Qwen, Antigravity,
Grok Build CLI, ZCode

### 리포트 모드

| 명령 | 내용 |
| --- | --- |
| `daily` / `weekly` / `monthly` | 기간별 집계 |
| `session` | 대화 세션별 집계 |
| `blocks` | Claude Code 5시간 과금 창 추적 (`--active`로 진행 중인 창만) |
| `statusline` | Claude Code 상태바용 압축 출력 (Beta) |

---

## 2. 폴더 구조 전수조사

| 위치 | 역할 |
| --- | --- |
| `rust/crates/` (8개) | 두뇌. `ccusage-core`(가격·집계·출력), `ccusage-cli`·`ccusage-cli-parser`(명령 파싱), `ccusage-terminal`(표 렌더링), `ccusage-config`(설정), `ccusage`(최종 바이너리), `ccusage-adapter-all`, `ccusage-test-support` |
| `rust/adapters/` (19개) | 눈. 에이전트별 로그 해석기 + `common`(파일 walking, size-balanced chunking, 병렬 읽기) |
| `apps/ccusage/` | npm 배포 표면. 실질 코드는 `src/cli.js`(OS 감지 후 네이티브 바이너리 spawn) + `config-schema.json` |
| `packages/ccusage-*` (6개) | OS×아키텍처별 네이티브 바이너리 패키지 (darwin/linux/win32 × arm64/x64). `optionalDependencies`로 설치됨 |
| `docs/` | VitePress 문서 43개 → ccusage.com |
| `.agents/skills/` (17개) | **개발용(AI용) 내부 규칙집**: `rust`, `typescript`, `testing`, `tdd`, `commit`, `create-pr`, `fix-ci`, `docs`, `profile`, `agent-sources` 등 |
| `.github/workflows/` (11개) | CI, 릴리스(tagpr), 가격 데이터 자동 갱신 PR, PR/이슈 게이트, 링크 체크 |
| `flake.nix` / `justfile` | Nix 재현 빌드 + `just` 명령 러너 (`just test`, `just check`, `just fmt`) |

규모: Rust 소스 152개 파일 / 약 63,000줄. 모노레포 버전 `20.0.22`.

### 아키텍처 포인트

- **어댑터 패턴**: 소스가 18개여도 공통 리포트 형태로 흡수. 소스별 특수 로직은
  `rust/adapters/<agent>`, 공통 로직은 `ccusage-core` / `ccusage-adapter-common`.
- **성능**: 로그가 GB급까지 커지므로 Rust 병렬 파싱. 그래서 상태바에 실시간 노출 가능.
- **중복 제거**: Codex 어댑터의 `replay.rs`는 포크·리플레이된 세션의 토큰 중복 집계를 방지.
- **오프라인 폴백**: 가격표를 `include_bytes!`로 바이너리에 압축 내장 → `--offline`에서 네트워크 0.

---

## 3. 주요 질문 답변

### 설치 및 사용법

```bash
# 설치 없이 실행 (권장)
npx ccusage@latest
bunx ccusage            # bun은 캐시되어 재실행이 빠름
pnpm dlx ccusage

# 전역 설치
npm install -g ccusage

# Nix
nix run github:ccusage/ccusage -- daily
```

자주 쓰는 명령:

```bash
ccusage                       # 전체 소스 일별 (기본)
ccusage daily / weekly / monthly / session
ccusage blocks --active       # 진행 중인 5시간 창
ccusage claude daily
ccusage codex daily --speed fast

ccusage daily --since 2026-04-25 --until 2026-05-16
ccusage daily --last 1        # 오늘
ccusage daily --json          # JSON 출력
ccusage daily --breakdown     # 모델별 분해
ccusage daily --by-agent --json
ccusage daily --no-cost       # 비용 숨김(공유용)
ccusage --compact             # 스크린샷용 좁은 표
ccusage daily --timezone Asia/Seoul
ccusage daily --offline       # 네트워크 없이 내장 가격표 사용
ccusage claude daily --instances --project myproject
```

설정 파일(`ccusage.json`)로 기본값 고정 가능하며, `config-schema.json`이 있어
에디터 자동완성·검증이 된다.

상태바 연동 — `~/.claude/settings.json`:

```json
{
  "statusLine": { "type": "command", "command": "bun x ccusage statusline", "padding": 0 }
}
```

### 플러그인? 스킬? MCP?

**세 가지 모두 아니다. 독립 CLI 프로그램(npm 런처 + Rust 네이티브 바이너리)이다.**

| 구분 | 해당 여부 | 비고 |
| --- | --- | --- |
| CLI 도구 | ✅ | 본질 |
| Claude Code 플러그인 | ❌ | 플러그인 규격 없음 |
| Skill | ❌ (제품으로는) | `.agents/skills/`는 ccusage를 **개발**할 때 AI가 읽는 내부 문서 |
| MCP 서버 | ❌ | Rust 소스에 MCP 코드 0건. 루트 `.mcp.json`은 개발 보조(context7, grep)용 |
| Statusline 훅 | 🔶 | `statusline` 명령이 Claude Code statusLine 훅으로 통합 |

→ MCP 서버로 감싸는 것은 `--json` 출력을 tool로 노출하면 되므로 쉽고, 아직 빈 자리다.

### API 토큰이 필요한가

**필요 없다.** API 키·로그인·OAuth·결제 전부 불필요.

가격 데이터는 인증 없는 공개 JSON에서 받는다 (`rust/crates/ccusage-core/src/pricing.rs`):

- `https://raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json`
- `https://models.dev/api.json`

게다가 스냅샷이 바이너리에 내장되어 있어 `--offline`이면 네트워크 통신조차 없다.
대화 내용은 어떤 경우에도 외부로 나가지 않으므로 보안이 엄격한 환경에서도 쓸 수 있다.

### 왜 GitHub에서 유명한가

1. **타이밍** — Claude Code 확산기에 "내가 얼마 쓰는지 모르겠다"는 불안을 정확히 해결.
2. **바이럴 구조** — 사용량 스크린샷 공유가 자연스럽다. `--compact`가 애초에 공유용 옵션.
3. **마찰 0** — `npx ccusage@latest` 한 줄로 10초 만에 결과.
4. **커버리지** — 18개 에이전트 지원으로 "AI 코딩 비용 표준 도구" 포지션 선점.
5. **프라이버시** — 로컬 전용, 키 불필요 → 도입 결정이 쉽다.
6. **품질** — Rust 워크스페이스, Nix 재현 빌드, 11개 CI, 가격표 자동 갱신, 스폰서(CodeRabbit,
   Blacksmith, Lineman.io) 확보.
7. **생태계** — macOS 메뉴바 앱, Raycast 익스텐션, Neovim 플러그인, 웹 대시보드, 랭킹 사이트
   (viberank, CCWarriors, Token Battle, Straude)까지 파생. `--json` 하나가 생태계를 만들었다.

### 로컬 에이전트 구축에 도움이 되는가

**도움이 된다. 세 가지 층위로 활용 가능하다.**

1. **도구로 즉시 활용** — 비용 관측, 프롬프트 개선 전후 토큰 회귀 측정,
   `ccusage blocks --active --json` 폴링으로 한도 가드레일, `--breakdown`으로 모델 라우팅 근거.
2. **아키텍처 교본** — 어댑터 패턴, 대용량 JSONL 스트리밍/병렬 파싱, 세션 포크 중복 제거,
   오프라인 폴백, `.agents/skills/` 방식의 AI 규칙 문서화.
3. **데이터 소스로 통합** — `ccusage session --json`을 컨텍스트로 넣어 자기 사용량을 회고하는
   에이전트, 주간 비용 리포트 에이전트 구성.

한계: 로그 파일 기반 **사후 분석** 도구라 수 초~수십 초 지연이 있고, 로그를 남기지 않는
커스텀 에이전트는 어댑터를 직접 작성해야 한다.

### React나 PHP로 만들 수 있는가

**완전 재구현은 비추천, 위에 올리는 것은 강력 추천.**

재구현이 나쁜 이유: (1) GB급 JSONL 파싱 성능, (2) 18개 로그 포맷이 계속 변해 추적 비용이 큼,
(3) 가격표 동기화 파이프라인을 새로 만들어야 함, (4) MIT라 그냥 쓰면 됨.

권장 구조:

```
ccusage --json  →  백엔드(PHP/Node)  →  프론트(React)
   (엔진)            (수집·정산)          (시각화)
```

- **React**: Electron 또는 **Tauri** 데스크톱 앱. Tauri는 Rust 기반이라 ccusage 크레이트를
  라이브러리로 직접 링크하는 것도 가능. 또는 사용자가 `ccusage monthly --json`을 업로드하는
  웹앱(viberank·Token Battle 방식).
- **PHP**: 파서가 아니라 **수집·저장·리포팅 레이어**로. `shell_exec('ccusage daily --json')`을
  cron으로 돌려 중앙 DB에 적재 → Laravel + Filament 관리 패널 → 월별·부서별 정산서.

| 접근 | 강점 | 권장 역할 |
| --- | --- | --- |
| React | 실시간 시각화, 데스크톱 앱, 인터랙션 | 프론트 / 앱 셸 |
| PHP | 서버 수집, 팀 집계, 정산·권한 | 백엔드 / 수집기 / 관리자 |

---

## 4. 수익화 아이디어

### 시장 검증 근거

- README의 메인 스폰서 **Lineman.io**가 "Teams & Enterprise cost monitoring, 40% lower token
  usage, unauthorized-spend alerts"로 **이미 유료 사업 중**이며 ccusage에 스폰서 비용을 쓴다.
- 커뮤니티 문서에 랭킹/공유 사이트가 4개 존재 → 사용자가 사용량을 비교·자랑하려는 수요가 있다.

### 아이디어 1. 팀 AI 비용 관리 SaaS (수익 최상 / 난이도 상)

각 개발자 PC에서 `ccusage daily --json --by-agent`를 cron으로 수집(숫자만 전송) → 중앙 서버에서
팀·부서·프로젝트별 집계, 예산 소진율, 이상 감지, Slack/Teams 알림, 월말 정산서 PDF, 벤더 비교
리포트 제공.

| 플랜 | 가격 |
| --- | --- |
| Free | $0 (3인까지) |
| Team | $8 / seat / 월 |
| Business | $15 / seat / 월 (SSO, 감사로그, 예산 강제) |
| Enterprise | 협의 ($2,000+/월) |

50인 고객 = 월 $400. 고객 25곳이면 월 $10,000 수준. 핵심 메시지는
**"코드·프롬프트는 보지 않고 숫자만 받는다"** — ccusage 구조상 기술적으로 참이므로 보안 심사가 쉽다.

### 아이디어 2. 한국 시장 특화 대시보드 (가성비 최상 / 난이도 중)

원화 실시간 환산, **세금계산서·경비처리 리포트**, 카카오톡 알림봇, 네이버웍스·잔디 연동,
한국어 UI/지원. 글로벌 경쟁자가 하지 않는 로컬라이제이션이 진입장벽이 된다.

- 개인 월 4,900원 / 팀 1인당 월 9,900원 + 세금계산서 발행 대행.

### 아이디어 3. 데스크톱 앱 유료 Pro (난이도 중)

무료는 기본 표·최근 7일, Pro($5/월 또는 평생 $39)는 무제한 히스토리, 트렌드 예측,
5시간 블록 위젯·알림, 에이전트/프로젝트 비교 차트, CSV·PDF 내보내기.
Tauri로 만들면 단일 실행 파일 + 초고속. 결제는 Gumroad / Lemon Squeezy.

### 아이디어 4. AI 원가 청구 자동화 (숨은 보석 / 난이도 중하)

`ccusage claude daily --instances --project <name> --json`이 프로젝트별 분리를 이미 지원한다.
여기에 프로젝트↔클라이언트 매핑, 마크업(원가 × 1.3), 인보이스 PDF 자동 생성, 회계·세금계산서
연동을 얹는다. 타깃은 프리랜서·소규모 에이전시. 월 $12 또는 인보이스 금액의 1%.
세일즈 포인트: "월 $12로 월 $300 회수".

### 아이디어 5. MCP 서버 + 최적화 컨설팅 (수익성 최상)

`ccusage --json`을 MCP tool로 노출해 Claude Code 안에서 대화로 비용을 조회·분석하게 한다
(현재 ccusage에 MCP가 없어 선점 가능). 무료 배포로 유입을 만들고, 프롬프트 최적화·모델 라우팅
설계 컨설팅(건당 200~500만원, 리테이너 월 100만원)으로 전환한다.

### 아이디어 6. 랭킹·커뮤니티 + 광고/스폰서 (난이도 하)

한국 커뮤니티용 랭킹 서비스는 아직 없다. 바이럴 계수가 높아 아이디어 1·2로 유입시키는
깔때기 상단으로 쓸 수 있다.

### 추천 로드맵

```
1단계 (1~2주)   원화 환산 + 카톡 알림 도구를 무료 공개 → 반응 검증 (리스크 0)
2단계 (1~2개월) 아이디어 2 또는 4를 유료화 → 첫 매출과 피드백
3단계 (3~6개월) 아이디어 1 팀 SaaS로 확장 → B2B 진입
병행           MCP 서버 무료 배포 = 마케팅 + 컨설팅 리드 확보
```

라이선스가 MIT라 상업적 활용이 자유롭고(출처 표기 준수), 가장 어려운 수집·파싱을 ccusage가
대신해 주므로 우리는 UI·정산·알림 레이어에 집중할 수 있다.

---

## 5. 참고 링크

| 항목 | 주소 |
| --- | --- |
| 이 포크 저장소 | <https://github.com/bmshin94/ccusage> |
| 원본 저장소 | <https://github.com/ccusage/ccusage> |
| 공식 문서 | <https://ccusage.com> |
| npm | <https://www.npmjs.com/package/ccusage> |
| DeepWiki (코드 해설) | <https://deepwiki.com/ccusage/ccusage> |
| 원작자 | <https://github.com/ryoppippi> |
| 스폰서 | <https://github.com/sponsors/ryoppippi> |
| 가격 데이터 출처 (LiteLLM) | <https://github.com/BerriAI/litellm> |
| 가격 데이터 출처 (models.dev) | <https://models.dev> |
| 커뮤니티 프로젝트 목록 | `docs/guide/community-projects.md` |
