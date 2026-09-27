# OpenBrief 전수조사 분석 및 수익화 전략 (한국어)

> 이 문서는 OpenBrief 저장소를 전수조사한 결과와, 활용·수익화 방안에 대한
> 논의 내용을 정리한 기록입니다.
>
> - 작성일: 2026-09-27
> - 분석 대상 저장소: <https://github.com/bmshin94/openbrief>
> - 원본(업스트림) 저장소: <https://github.com/tantara/openbrief>
> - 데모 영상: <https://youtu.be/OnS3EViayRo>

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [저장소 구조 전수조사](#2-저장소-구조-전수조사)
3. [아키텍처 분석](#3-아키텍처-분석)
4. [지원 AI 모델](#4-지원-ai-모델)
5. [쉬운 설명 (비유 중심)](#5-쉬운-설명-비유-중심)
6. [Q&A: 설치 및 사용법](#6-qa-설치-및-사용법)
7. [Q&A: 플러그인/스킬/MCP 여부](#7-qa-플러그인스킬mcp-여부)
8. [Q&A: API 토큰 필요 여부](#8-qa-api-토큰-필요-여부)
9. [Q&A: GitHub에서 주목받는 이유](#9-qa-github에서-주목받는-이유)
10. [Q&A: 로컬 에이전트 구축 활용도](#10-qa-로컬-에이전트-구축-활용도)
11. [Q&A: React / PHP 구현 가능성](#11-qa-react--php-구현-가능성)
12. [수익화 아이디어 상세](#12-수익화-아이디어-상세)
13. [실행 로드맵](#13-실행-로드맵)
14. [참고 자료](#14-참고-자료)

---

## 1. 프로젝트 개요

### 한 줄 정의

OpenBrief는 **영상/오디오/PDF를 가져와 전사(transcript)하고, 근거 기반 요약을
생성하고, 그 내용으로 채팅하고, 요약을 음성으로 들을 수 있는 크로스플랫폼
데스크톱 애플리케이션**이다.

성격상 "NotebookLM 또는 유튜브 요약 서비스의 오픈소스 · 로컬 실행 버전"에 해당한다.

### 기본 정보

| 항목 | 내용 |
|---|---|
| 버전 | `0.4.0` (`client/apps/tauri/package.json`, `src-tauri/Cargo.toml`) |
| 라이선스 | **AGPL-3.0-only** |
| 프레임워크 | Tauri v2 (Rust) + React 19 + TypeScript |
| 대상 플랫폼 | macOS / Windows / Linux |
| 저장소 형태 | pnpm + Turborepo 모노레포 |
| 전체 파일 수 | 479개 (`.git` 제외) |
| 패키지 매니저 | pnpm `11.0.9` (Node `^22.21.0`) |

### 핵심 기능

- 영상 링크 붙여넣기 또는 로컬 오디오/영상 파일 가져오기
- 유튜브 자막 추출 또는 온디바이스 STT(음성인식)
- 타임스탬프가 포함된 블로그형 마크다운 브리핑 생성
- 요약 또는 전체 전사문을 근거로 한 질의응답 채팅
- TTS로 요약을 음성으로 재생
- 로컬 실행 기반 프라이버시 보장, 오픈소스 무료

---

## 2. 저장소 구조 전수조사

### 루트

```text
openbrief/
├── AGENTS.md          AI 에이전트용 개발 규칙서 (아키텍처/검증/커밋 규칙)
├── CLAUDE.md          프로젝트 페르소나 가이드
├── README.md          제품 소개 + 셋업 가이드
├── LICENSE            AGPL v3 전문
├── docs/
│   ├── LOCAL_MODEL.md      로컬 모델 체크포인트 저장 규칙
│   ├── release.md          릴리즈 절차
│   ├── WINDOWS_SIGNING.md  윈도우 코드사인
│   └── assets/             스크린샷, 썸네일, 데모 음성
├── .github/
│   ├── ISSUE_TEMPLATE/     버그/기능 요청 템플릿
│   └── workflows/          CI 6종 (전부 workflow_dispatch 수동 실행)
│       ├── release.yml                 플랫폼별 빌드/배포
│       ├── transcribe-smoke.yml        전사 스모크 테스트
│       ├── qwen-asr-smoke.yml          Qwen3-ASR 스모크
│       ├── qwen-tts-smoke.yml          Qwen3-TTS 스모크
│       ├── supertonic-smoke.yml        Supertonic TTS 스모크
│       └── bundled-binary-smoke.yml    번들 바이너리 검증
└── client/            실제 코드 본진
```

### `client/apps` — 애플리케이션 5종

| 앱 | 역할 | 완성도 |
|---|---|---|
| **`tauri/`** | 메인 데스크톱 앱. 코드 대부분이 여기에 있다 | 높음 |
| `nextjs/` | 마케팅 홈, 다운로드 페이지, 유튜브 업로드/랭킹/대시보드, 공유 페이지, 피드백 | 작동 |
| `tanstack-start/` | TanStack Start 앱 셸 | 셸 수준 |
| `expo/` | React Native 앱 셸 | 셸 수준 |
| `workers/` | Cloudflare Worker (유튜브 스크린샷 추출) | 보조 |

### `client/packages` — 공용 패키지 7종

| 패키지 | 내용 |
|---|---|
| `api/` | tRPC 라우터 (`share`, `youtube`, `auth`, `post`, `feedback`) |
| `auth/` | better-auth 통합 (Discord OAuth 설정 예시 포함) |
| `db/` | Drizzle ORM + Supabase Postgres 스키마 |
| `ui/` | shadcn/ui 기반 공용 컴포넌트 18종 |
| `validators/` | zod 검증 + HMAC 서명 helper |
| `model-card/` | AI 모델 메타정보 카드 |
| `openbrief-content/` | 공용 콘텐츠/문구 |

### `client/tooling` — 공용 설정

`eslint/`, `prettier/`, `tailwind/`, `typescript/`, `github/`

---

## 3. 아키텍처 분석

### 3.1 프론트엔드 (`client/apps/tauri/src`)

#### features — 화면 단위 (라인 수 기준 중요도)

| 화면 | 라인 수 | 역할 |
|---|---|---|
| `workbench/WorkbenchView.tsx` | 4,309 | 메인 작업실. 플레이어 + 전사문 + 요약 + 채팅 |
| `settings/SettingsView.tsx` | 2,007 | AI 제공자, 모델, API 키, 테마, TTS 설정 |
| `finder/FinderView.tsx` | 1,258 | 라이브러리 (가져온 자산 목록) |
| `setup/SetupDialog.tsx` | 748 | 초기 셋업 위저드 (모델 다운로드 등) |
| `voices/VoicesView.tsx` | 490 | TTS 목소리 선택/미리듣기 |
| `playlists/PlaylistView.tsx` | 461 | 플레이리스트 관리 |
| `onboarding/`, `tutorial/`, `faq/` | 75~153 | 온보딩/튜토리얼/FAQ |
| `transcript-overlay/` | 85 | 항상 위에 뜨는 실시간 자막 오버레이 창 |

#### domain — 순수 로직 (각 파일에 `.test.ts` 동반)

`chat`, `ingest`, `summary`, `transcript`, `transcript-actions`, `provider`,
`podcast`, `quiz`, `share`, `settings`, `media-library`, `markdown-save`,
`compatibility`, `platform`, `download-error`, `helper-protocol`

#### services — 부수효과 (34개 파일)

전사, 요약채팅, TTS(Supertonic), 아티팩트 내보내기, 워크스페이스, 미디어
라이브러리 저장소, 클립보드, 파일 다이얼로그, 플랫폼 플러그인, 온보딩 상태,
시스템 프롬프트 설정, 테마 설정, 런타임 로거 등

#### i18n — 15개 언어

`ko_kr`, `en_us`, `ja_jp`, `zh_cn`, `zh_tw`, `de_de`, `es_es`, `fr_fr`,
`it_it`, `pt_br`, `ru_ru`, `uk_ua`, `pl_pl`, `el_gr`, `ar_ma`

### 3.2 백엔드 (`client/apps/tauri/src-tauri/src`)

Rust가 신뢰 경계(trusted boundary)를 담당한다. Tauri command 43개 노출.

| 모듈 | 역할 |
|---|---|
| `credentials.rs` | API 키를 OS 키체인에 저장, 실패 시 권한 0600 파일로 폴백 |
| `provider.rs` | LLM API 호출 대행 (일반 + 스트리밍) |
| `media_tools.rs` | yt-dlp 관리 및 자동 업데이트 정책 |
| `helper_sidecar.rs`, `helper.rs` | 헬퍼 프로세스 실행 (argv 배열만 사용) |
| `stt_models.rs` | STT 모델 카탈로그 + 다운로드 |
| `qwen_asr.rs` | Qwen3-ASR 음성인식 |
| `fluidaudio.rs` | Parakeet 음성인식 (Apple 플랫폼) |
| `supertonic.rs` | Supertonic 3 TTS (요약/채팅/팟캐스트 음성) |
| `trusted_paths.rs` | 파일 경로 권한 검증, 라이브러리 밖 접근 차단 |
| `media_library.rs` | 라이브러리 스냅샷 로드/저장 (rusqlite) |
| `workspace.rs` | 워크스페이스 생성/전환 |
| `ingest.rs` | URL/로컬 파일 가져오기 계획 |
| `storage_usage.rs` | 디스크 사용량 |
| `headless_download.rs` | CLI 모드 다운로드 |
| `platform_plugins.rs` | 플랫폼별 플러그인 계약 |
| `tests/config_security.rs` | 보안 설정 테스트 |

주요 Rust 의존성: `tauri 2`, `reqwest 0.13`, `rusqlite 0.32(bundled)`,
`transcribe-rs 0.3.8(whisper-cpp)`, `hound`, `subtp`, `futures-util`,
`tauri-plugin-{updater,shell,dialog,os,log,cli,single-instance}`

### 3.3 보안 설계 (핵심 차별점)

`credentials.rs`의 계약이 보안 불변식을 타입으로 강제한다.

```rust
CredentialStorageContract {
    owner: "tauri-rust",
    preferred_store: "os-keychain",
    fallback_store: "app-private-0600-file",
    renderer_receives_secret_values: false,
    helper_receives_secret_values: false,
}
```

- 렌더러(React)는 시크릿 원문을 절대 받지 않는다
- 헬퍼 프로세스도 시크릿 원문을 받지 않는다
- `redact_secret_fields` 명령이 `apikey`, `api_key`, `authorization`,
  `credential`, `oauth`, `refresh_token`, `secret`, `token`, `x-api-key`
  등을 포함한 필드를 자동 마스킹
- 헬퍼 실행은 셸 문자열 연결이 아니라 argv 배열만 사용
- 파일 경로는 라이브러리 상대경로로만 검증/저장

### 3.4 자기완결형 라이브러리 구조 (`AGENTS.md` 규칙)

```text
<라이브러리 루트>/
  videos/{id}/   원본 미디어, 전사문, 전사 변형, 요약, 채팅 세션,
  audios/{id}/   생성 TTS 오디오, 썸네일/프리뷰, 메타데이터, 번들 manifest
  pdfs/{id}/
```

설계 원칙: **각 자산 디렉터리는 zip으로 압축해 다른 기기에서 그대로 가져올 수
있는 이식 가능한 번들이어야 한다.** 그래서 manifest의 모든 아티팩트 참조는
라이브러리 상대경로로 저장된다. `transcripts/`, `summaries/` 같은 전역 버킷은
금지된다.

---

## 4. 지원 AI 모델

### 음성 → 텍스트 (STT)

| 엔진 | 비고 |
|---|---|
| Whisper | `whisper.cpp` / `transcribe-rs` 크레이트 |
| Parakeet TDT 0.6B v3 | FluidAudio, Apple 플랫폼 CoreML |
| Qwen3-ASR + ForcedAligner | |

### 텍스트 → 음성 (TTS)

Supertonic 3, Qwen3-TTS

### LLM (요약/채팅) — 6종

| 제공자 키 | 엔드포인트 |
|---|---|
| `openai` | `https://api.openai.com/v1/chat/completions` |
| `anthropic` | `https://api.anthropic.com/v1/messages` |
| `gemini` | `https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` |
| `openrouter` | `https://openrouter.ai/api/v1/chat/completions` |
| `deepseek` | `https://api.deepseek.com/v1/chat/completions` |
| `openai-compatible` | `http://localhost:1234/v1/chat/completions` (LM Studio 등 로컬) |

### LLM 작업 종류 (`ProviderOperation`) — 7가지

`summary`, `chat`, `podcast_script`, `quiz`, `transcript_review`,
`transcript_translate`, `transcript_resegment`

즉 README에 강조되지 않았지만 **퀴즈 생성, 팟캐스트 대본 생성, 전사 교정,
번역, 문장 재분할** 기능이 이미 구현되어 있다.

작업별 기본 생성 파라미터 예시:

| operation | temperature | topP | maxTokens |
|---|---|---|---|
| summary | 0.3 | 0.9 | 4096 |
| chat | 0.2 | 0.9 | 2048 |
| podcast_script | 0.55 | 0.95 | 4096 |
| quiz | 0.35 | 0.9 | 4096 |
| transcript_review | 0.1 | 0.9 | 4096 |
| transcript_translate | 0.1 | 0.9 | 4096 |
| transcript_resegment | 0.2 | 0.9 | 4096 |

---

## 5. 쉬운 설명 (비유 중심)

### 사용 흐름을 대화로 표현

```text
사용자: (유튜브 링크 붙여넣기)
OpenBrief: 영상 다운로드 (yt-dlp)
OpenBrief: 말한 내용을 전부 글로 옮김 (로컬 음성인식)
OpenBrief: 핵심 요약 + 타임스탬프 생성 (LLM)
사용자: "3번째 내용 더 설명해줘"
OpenBrief: "43분 12초 구간에 나옵니다" (근거 기반 채팅)
사용자: "요약본 읽어줘"
OpenBrief: TTS 음성 재생
```

### 공장 비유

```text
   [입구]                [작업장]                    [출구]
 유튜브 링크 ─┐
 mp4/mp3 파일 ─┼─> ① 다운로드 ─> ② 음성→글자 ─> ③ AI요약 ─> 요약문/채팅/음성
 PDF        ─┘    (yt-dlp)      (로컬 STT)     (LLM API)
```

- ① 다운로드: `yt-dlp` (번들)
- ② 받아쓰기: Whisper / Parakeet / Qwen3-ASR — **로컬 실행, 외부 전송 없음**
- ③ 요약: GPT / Claude / Gemini / DeepSeek / 로컬 LLM — 외부 API 호출
- ④ 읽어주기: Supertonic 3 / Qwen3-TTS

### 웹 서비스와의 차이

| | 일반 웹 서비스 | OpenBrief |
|---|---|---|
| 형태 | 브라우저 접속 | PC 설치형 앱 |
| 내 파일 | 서버 업로드 필요 | 내 컴퓨터에 그대로 |
| 음성인식 | 서버가 수행 | 내 PC가 수행 |
| 비용 | 월 구독료 | 무료 (LLM API 비용만) |
| 오프라인 | 불가 | 전사·TTS는 가능 |
| 데이터 소유 | 서비스 종료 시 소실 | 내 폴더에 영구 보관 |

### 보안 구조를 집으로 비유

```text
┌─────────────────────────────────────┐
│  거실 (React 렌더러)                 │
│    "이 내용 요약해줘" 요청만 전달       │
│  ══════ 방화문 (Tauri IPC) ══════   │
│  금고방 (Rust)                       │
│    - API 키 실물 보관 (OS 키체인)     │
│    - 실제 네트워크 통신               │
│    - 파일 경로 권한 검증              │
└─────────────────────────────────────┘
```

---

## 6. Q&A: 설치 및 사용법

### 6.1 그냥 쓰기만 할 경우

GitHub Releases 또는 다운로드 페이지에서 설치 파일을 받는다
(macOS `.dmg` / Windows `.msi` / Linux `.AppImage`).
`client/apps/nextjs/src/app/download/page.tsx`가 릴리즈 링크를 직접 제공한다.

### 6.2 소스 빌드 (개발자)

**준비물**

| 항목 | 버전 |
|---|---|
| Node.js | `^22.21.0` |
| pnpm | `11.0.9` |
| Rust + Cargo | 최신 |
| Tauri v2 사전조건 | macOS: Xcode CLI / Windows: VS Build Tools + WebView2 / Linux: webkit2gtk 등 |

**설치**

```bash
git clone https://github.com/bmshin94/openbrief
cd openbrief/client
pnpm install
```

pnpm이 native 빌드 스크립트를 무시했다고 경고하면 `pnpm approve-builds`로
승인 후 `pnpm install`을 다시 실행한다.

**환경변수 (웹앱 전용, 데스크톱만 쓸 경우 불필요)**

```bash
cp .env.example .env
```

`.env.example` 항목: `POSTGRES_URL`, `AUTH_SECRET`, `AUTH_DISCORD_ID`,
`AUTH_DISCORD_SECRET`, `WORKER_BASE_URL`, `WORKER_SHARED_SECRET`,
`R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`

**실행**

```bash
# 데스크톱 앱 (메인)
cd client && pnpm dev:tauri
#   헬퍼 사이드카 빌드 -> Vite(:1420) -> Rust 컴파일 -> 앱 창

# 웹앱
cd client && pnpm dev:next   # http://localhost:3000

# 전체 dev 태스크
cd client && pnpm dev
```

**사이드카 / 미디어 도구 준비**

```bash
cd client/apps/tauri
pnpm setup:dev-sidecars     # helper + fluidaudio + supertonic + localai + 서명
pnpm prepare:media-assets   # yt-dlp / ffmpeg / ffprobe
```

**검증**

```bash
cd client/apps/tauri && pnpm test:run     # vitest
cd client/apps/tauri && pnpm typecheck    # tsc --noEmit
cd client/apps/tauri/src-tauri && cargo check
cd client && pnpm lint && pnpm build
git diff --check
```

**DB / 인증 helper**

```bash
cd client
pnpm db:push
pnpm db:studio
pnpm auth:generate
```

### 6.3 앱 사용 흐름

1. 앱 실행 → `OnboardingView` → `SetupDialog` (STT 모델 다운로드, AI 제공자/API 키 설정)
2. `FinderView`(라이브러리) → 추가 → `AddVideoDialog`에 링크 붙여넣기 또는 파일 드래그앤드롭
3. 항목 클릭 → `WorkbenchView` (플레이어 / 전사문 / 요약·채팅 탭)
4. 요약 생성 → 마크다운 브리핑 (TipTap 에디터로 직접 편집 가능)
5. 채팅 → 전사문·요약 근거로 질의응답
6. 읽어주기 → TTS 음성 생성
7. 내보내기 → 마크다운/텍스트 아티팩트 저장
8. 추가: 플레이리스트, 실시간 자막 오버레이 창, 워크스페이스 전환

---

## 7. Q&A: 플러그인/스킬/MCP 여부

**결론: 셋 다 아니다. 독립 실행형 데스크톱 애플리케이션이다.**

| 구분 | 정의 | OpenBrief |
|---|---|---|
| Plugin | 다른 프로그램에 끼워넣는 확장 | 아님 |
| Skill | Claude에 주는 지시서 폴더 (`SKILL.md`) | 아님 |
| MCP | AI가 외부 도구를 쓰게 하는 표준 프로토콜 | 아님 |
| Standalone App | 독립 설치·실행되는 프로그램 | **해당** |

**근거**

- `src-tauri/Cargo.toml`이 독립 실행 바이너리 `openbrief`를 생성
- `tauri-plugin-updater` = 자체 자동 업데이트 → 스스로 배포되는 앱
- `tray-icon`, `single-instance` = 트레이 상주 네이티브 앱
- MCP SDK 의존성 없음, `SKILL.md` 없음

`tauri-plugin-opener` 등은 Tauri 프레임워크 내부 모듈이며, OpenBrief가
*사용하는* 플러그인이다. OpenBrief 자체가 플러그인인 것은 아니다.

**방향이 반대**: OpenBrief는 AI의 플러그인이 아니라, **AI를 도구로 호출하는
앱**이다. `AGENTS.md`와 `CLAUDE.md`는 "AI가 이 코드를 수정할 때 지킬 규칙"
문서로, MCP나 Skill과 역할이 다르다.

다만 이 앱의 기능(전사/요약/채팅)을 MCP 서버로 감싸면 Claude 등이 OpenBrief를
도구로 사용할 수 있게 만들 수 있다. 이는 수익화 아이디어 ①에 해당한다.

---

## 8. Q&A: API 토큰 필요 여부

| 기능 | API 키 | 필요 조건 |
|---|---|---|
| 영상/오디오 다운로드 | 불필요 | yt-dlp (번들) |
| 유튜브 자막 추출 | 불필요 | — |
| 로컬 STT (Whisper/Parakeet/Qwen3-ASR) | 불필요 | 모델 다운로드만 |
| 로컬 TTS (Supertonic 3) | 불필요 | — |
| AI 요약 | **필요** | LLM 제공자 API 키 |
| AI 채팅 | **필요** | LLM 제공자 API 키 |
| 퀴즈 / 팟캐스트 대본 / 번역 / 전사 교정 | **필요** | LLM 제공자 API 키 |
| 웹앱 로그인·공유·업로드 | 필요 | Supabase, Discord OAuth, R2 |

### API 키 없이 쓰는 방법

`openai-compatible` 제공자의 기본 엔드포인트가 `http://localhost:1234`
(LM Studio 기본 포트)이고, 기본 모델 목록에 `llama-3.2-3b-instruct`가 포함되어
있다. 따라서 LM Studio / Ollama / vLLM으로 로컬 모델을 띄우면
**API 비용 0원 + 완전 오프라인**으로 사용할 수 있다.

### 유료 API 비용 감각

| 제공자 | 특징 |
|---|---|
| DeepSeek | 가장 저렴, 가성비 |
| OpenRouter | 여러 모델을 한 키로 |
| Gemini | 무료 티어 넉넉 (기본값이 `gemini-3.1-flash-lite`인 이유) |
| OpenAI / Anthropic | 품질 우수, 상대적 고비용 |

기본 모델이 전반적으로 저비용 티어로 설정되어 있고, `maxTokens`도 작업별로
제한되어 있어 비용을 배려한 설계다.

### API 키 저장 위치

1순위 OS 키체인 (macOS Keychain / Windows Credential Manager),
2순위 앱 전용 디렉터리의 권한 0600 파일.
렌더러와 헬퍼는 키 원문을 받지 못하고, 로그는 자동 마스킹된다.

---

## 9. Q&A: GitHub에서 주목받는 이유

> 주의: 원본 저장소의 실제 스타 수는 본 분석 환경에서 확인하지 못했다.
> 아래는 코드·README·시장 포지셔닝에 근거한 분석이다.

1. **타이밍** — NotebookLM류 서비스의 인기로 "로컬에서 쓰고 싶다"는 수요가 큼.
   오픈소스 대안 카테고리는 확산 속도가 빠르다.
2. **프라이버시 + 로컬 우선** — 회의 녹음을 외부 서버에 올리기 어려운 개인·기업
   수요를 직접 해결.
3. **개발자 취향 스택** — Tauri v2 + Rust + React 19 + TypeScript + Turborepo +
   shadcn/ui + tRPC + Drizzle. 코드 구경 목적의 스타를 부르는 조합.
4. **완성도** — 파일 479개, 도메인 로직마다 테스트, i18n 15개 언어,
   릴리즈 CI + 스모크 테스트 5종, 윈도우 코드사인 문서까지. "장난감이 아닌 제품".
5. **최신 모델 대응 속도** — Qwen3-ASR, Supertonic 3, Parakeet TDT 등을 빠르게 지원.
   해당 모델 커뮤니티에서 자연 유입.
6. **로컬 LLM 커뮤니티 흡수** — `openai-compatible` 하나로 LM Studio/Ollama
   사용자층을 포섭.
7. **크로스플랫폼 + 즉시 설치** — 3개 OS 릴리즈 제공으로 "빌드 실패로 못 씀"이 없음.
8. **문서/데모 품질** — 데모 영상, 스크린샷, 배지, 목차, 모델 지원 표,
   체크박스 로드맵을 갖춘 README.

---

## 10. Q&A: 로컬 에이전트 구축 활용도

**결론: 매우 높다. 에이전트의 "몸통"이 완성품으로 들어 있다.**

### 재활용 가치가 높은 부품

| 필요 기능 | 위치 | 가치 |
|---|---|---|
| 멀티 LLM 어댑터 | `services/providerAdapters.ts`, `domain/provider.ts`, `src-tauri/src/provider.rs` | 매우 높음 |
| 스트리밍 응답 | `provider.rs::complete_provider_stream_request` (`futures-util` + `reqwest` stream) | 매우 높음 |
| API 키 안전 저장 | `credentials.rs` (OS 키체인 + 0600 폴백) | 매우 높음 |
| 로컬 LLM 연결 | `openai-compatible` 제공자 + `build-localai-sidecar.js` | 매우 높음 |
| 로컬 STT 파이프라인 | `stt_models.rs`, `qwen_asr.rs`, `fluidaudio.rs` | 매우 높음 |
| 로컬 TTS 파이프라인 | `supertonic.rs` | 매우 높음 |
| 사이드카 프로세스 관리 | `helper_sidecar.rs` + `scripts/build-*-sidecar.js` | 매우 높음 |
| 모델 다운로드/카탈로그 | `stt_models.rs`, `docs/LOCAL_MODEL.md` (`.partial` 처리 포함) | 매우 높음 |
| 파일 권한 샌드박싱 | `trusted_paths.rs` | 매우 높음 |
| 근거 추적 | `EvidenceAnchor` 타입 (`domain/media-library.ts`) | 높음 |
| 채팅 세션 관리 | `domain/chat.ts` + rusqlite | 높음 |
| 작업별 파라미터 프리셋 | `defaultGenerationParamsByOperation` | 높음 |
| 시스템 프롬프트 관리 | `services/systemPromptSettingsService.ts` | 높음 |
| 워크스페이스 분리 | `workspace.rs` | 중간 |

### 배울 아키텍처 패턴

1. **3층 분리** — `domain`(순수 함수, 테스트 용이) / `services`(부수효과) /
   Rust(권한 필요 작업). 에이전트 코드가 스파게티가 되는 것을 막는다.
2. **요청을 데이터로 표현** — `ProviderRequestPlan`(provider, operation,
   endpoint, headers, body, credentialPolicy)을 순수 객체로 만들고 Rust가 실행.
   계획 생성은 테스트하기 쉽고 실행은 안전하다.
3. **operation 단위 추상화** — 에이전트의 "능력"을 열거형으로 정의하고 각각 다른
   프롬프트·파라미터를 부여. 툴/스킬 시스템의 기초 형태.
4. **보안 불변식을 타입으로 강제** — `renderer_receives_secret_values: false`
   처럼 "절대 일어나면 안 되는 일"을 계약으로 못 박는다.

### 없는 것 (직접 구현 필요)

- Tool Calling / Function Calling (텍스트 생성만 지원)
- 에이전트 루프 (계획 → 실행 → 관찰 → 반복)
- 벡터 DB / RAG 임베딩 (README 로드맵에 video embedding이 TODO)
- MCP 프로토콜
- 멀티 에이전트 오케스트레이션

즉 **"두뇌(추론 루프)"는 없지만 "몸통(LLM 연결·음성·파일·보안·모델 관리)"은
완성되어 있다.** 로컬 에이전트를 만든다면 이 구조를 껍데기로 쓰고 에이전트
루프만 추가하는 것이 최단 경로다.

---

## 11. Q&A: React / PHP 구현 가능성

### 11.1 React — 이미 React로 구현되어 있다

화면 전체가 React 19 + TypeScript + Vite + Tailwind + shadcn/ui +
TipTap(에디터) + lucide-react(아이콘)다. React를 다룰 수 있다면 UI는 바로
수정 가능하다.

**다만 브라우저 React만으로는 불가능한 부분**

| 기능 | 브라우저 React | 이유 |
|---|---|---|
| UI 전체 | 가능 | — |
| 유튜브 다운로드 | 불가 | yt-dlp = OS 프로세스 실행 필요 |
| 로컬 파일 읽기/쓰기 | 제한적 | 브라우저 샌드박스 |
| 로컬 Whisper 실행 | 매우 느림 | wasm/WebGPU 가능하나 실용성 낮음 |
| API 키 안전 저장 | 불가 | 브라우저에 OS 키체인 없음 |
| 트레이/오버레이 창 | 불가 | 브라우저 제약 |
| 자동 업데이트 | 불가 | 브라우저 제약 |

### 11.2 선택 가능한 구현 조합

| 방식 | 프론트 | 백엔드 | 난이도 | 특징 |
|---|---|---|---|---|
| A. Tauri (현재) | React | Rust | 높음 | 용량 작고 성능 최고 |
| B. Electron | React | Node.js | **낮음** | Rust 불필요, 용량 큼 |
| C. React + 로컬 서버 | React | PHP/Node/Python | 가장 낮음 | 배포가 번거로움 |
| D. 웹 서비스 | React | PHP/Node | 중간 | 서버비 + AGPL 리스크 |

Rust 학습 부담이 크면 **B안(Electron + Node.js)**이 현실적이다.
Node에서 `yt-dlp` 실행, `keytar`로 키체인 접근, `whisper.cpp` 바인딩으로 STT.

### 11.3 PHP — 가능하지만 역할이 다르다

**PHP로 가능한 것**

```php
// 외부 프로세스 실행 (yt-dlp / ffmpeg / whisper.cpp)
$cmd = ['yt-dlp', '-o', $out, $url];
proc_open(...);

// LLM API 호출 (Guzzle / curl)
// 파일 관리, MySQL, 큐, 인증, 결제
// 웹 UI (Laravel + Blade / Livewire)
```

**PHP로 불가능한 것**

- 데스크톱 앱 (트레이, 네이티브 창, 자동 업데이트)
- 사용자 PC에서의 오프라인 로컬 실행
- OS 키체인 접근

**현실적 정답: 하이브리드 웹 서비스**

```text
브라우저: React 또는 Livewire  (업로드 / 진행률 / 요약 / 채팅 UI)
    │ REST API
서버: PHP (Laravel)
    ├─ 업로드 / 인증 / 과금 (Cashier)
    ├─ Queue(Horizon) → 백그라운드 작업
    │     └─ proc_open: yt-dlp, ffmpeg
    │     └─ whisper.cpp 바이너리 또는 Whisper API
    ├─ LLM 호출 (Guzzle, 또는 prism-php 같은 멀티 LLM 라이브러리)
    └─ MySQL + S3/R2 저장
```

Laravel이 적합한 이유: Queue + Horizon(장시간 전사 작업에 필수),
Cashier(Stripe 결제), Sanctum(API 인증), 저렴한 배포 인프라.

한계 대처:

| 문제 | 해결 |
|---|---|
| PHP의 연산 성능 | STT는 `whisper.cpp` 바이너리에 위임, PHP는 오케스트레이션만 |
| 장시간 요청 타임아웃 | Queue + 폴링/SSE로 진행률 표시 |
| 실시간 스트리밍 | `response()->stream()` 또는 SSE |

### 11.4 권장 순서

1. **React + Laravel 웹 서비스** — 가장 빠른 수익화. 단 "로컬 프라이버시"라는
   OpenBrief의 최대 장점은 포기하고, "설치 없는 편의성 + 구독"으로 승부.
2. **Electron + React + Node.js 데스크톱** — 로컬/프라이버시 장점 유지,
   Rust 불필요. 라이선스 주의(코드 복제 대신 새로 작성).
3. **Tauri 포크** — 성능/용량 최고, Rust 학습 필요, AGPL 준수 필수.

---

## 12. 수익화 아이디어 상세

### 12.0 전제: 라이선스 전략 결정

```text
AGPL-3.0이 요구하는 것
  1) 수정 배포 시 수정 소스 공개
  2) 수정본을 네트워크 서비스로 제공 시에도 소스 공개
  3) 파생물도 AGPL (전염성)

AGPL이 금지하지 않는 것
  - 유료 판매 (가능)
  - 서비스 운영 (소스 공개하면 가능)
  - 프롬프트/모델/콘텐츠/데이터는 대상 아님
  - 별개 프로그램과의 API 통신은 파생물이 아님 (경계 설정은 신중히)
```

| 전략 | 내용 | 자유도 | 속도 |
|---|---|---|---|
| ① 준수 | AGPL 지키고 소스 공개 + 서비스로 수익 | 중 | 빠름 |
| ② 클린룸 | 코드 미사용, 아이디어/구조만 참고해 새로 작성 | **최고** | 느림 |
| ③ 프로세스 분리 | OpenBrief는 그대로 두고 별개 프로그램으로 감싸기 | 높음(법적 검토 필요) | 빠름 |

무료 오픈소스와 정면 경쟁하는 유료 제품은 어렵다. 특정 시장에 맞춰 새로 만드는
**클린룸 전략**이 라이선스 자유도와 제품 차별성 모두에서 유리하다.

### 12.1 아이디어 10선

#### ① MCP 서버 "미디어 이해 도구" — 시의성 최고

Claude / Cursor / ChatGPT가 영상을 이해하게 해주는 MCP 서버.

```text
transcribe_media(url)         영상·오디오 → 전사문
summarize_media(url, style)   요약 생성
search_transcript(url, q)     전사문 내 검색
get_timestamp(url, topic)     주제 등장 시각
extract_quotes(url)           인용문 추출
```

MCP 생태계가 빠르게 확장 중인데 "영상/오디오 이해" 카테고리는 아직 비어 있다.
OpenBrief가 파이프라인을 갖추고 있으므로 인터페이스만 얹으면 된다.

과금: 무료 100분/월 → Pro $15/월(2,000분) → Team $49/월 → API 종량제

| 난이도 | 초기비용 | 시장 | 차별성 |
|---|---|---|---|
| 낮음 | 낮음 | 확장 중 | 매우 높음 |

#### ② B2B 온프레미스 "사내 회의록 자동화"

회사 서버에 설치하는 회의록 시스템. Zoom/Teams 녹화 → 전사 → 요약 →
액션아이템 → Slack/Notion/Jira 전송.

세일즈 포인트: **"녹음 파일이 회사 밖으로 나가지 않는다."**
의료/법률/금융/공공/방산 등 클라우드 사용이 제한된 시장이 타깃.
AGPL은 오히려 감사 가능성(auditability) 측면에서 신뢰 자산이 된다.

| 항목 | 가격대 |
|---|---|
| 설치 + 커스터마이징 | 500만 ~ 3,000만원 (1회) |
| 연간 유지보수 | 라이선스의 20% |
| 좌석당 | 월 1만 ~ 3만원 |
| 교육/컨설팅 | 일 100만 ~ 200만원 |

기업은 소스가 아니라 설치·보증·지원에 비용을 지불한다. 수익 규모가 가장 크다.

#### ③ 버티컬 특화 SaaS

| 버티컬 | 특화 기능 | 가격 예시 |
|---|---|---|
| 의료 | 의학용어 사전, SOAP 노트, HIPAA | $99/월 |
| 법률 | 증언 녹취, 발화자 구분, 인용 추적 | $199/월 |
| 교육 | 강의노트 + 퀴즈 + 플래시카드 | $29/월 |
| 연구 | 인터뷰 질적분석, 코딩, 인용 | $49/월 |
| 팟캐스트 | 쇼노트, 챕터마커, SNS 클립, SEO | $39/월 |
| 영업 | 통화 분석, 이의제기 추적, CRM 연동 | $79/석 |
| 유튜버 | 타임스탬프, 설명란, 다국어 자막 | $19/월 |

기존 자산 활용도가 높다: `quiz` operation(교육), `podcast_script`(팟캐스트),
i18n 15개 언어(글로벌), `EvidenceAnchor`(법률·연구의 근거 추적).

#### ④ Managed 클라우드 (Open Core)

```text
무료       월 60분, 워터마크
Starter $9  월 600분
Pro    $29  월 3,000분 + API + 팀 공유
Business $99 무제한 + SSO + 감사 로그
```

AGPL 대처: 서버 코드는 공개하고 과금/멀티테넌시/관리 도구/인프라 코드는 별도
서비스로 분리. 다만 소스 공개 시 경쟁자 복제가 가능하므로 실제 상품은
호스팅·운영·지원이다. 인프라 비용과 경쟁이 부담.

#### ⑤ 플러그인/통합 마켓플레이스

본체는 무료 유지, 커넥터를 유료화: Notion / Obsidian / Slack / Teams / Jira /
Salesforce / Google Drive / Dropbox / Zapier / Zoom / Anki.
개별 $5/월 또는 전체 팩 $19/월.
커넥터는 별개 프로그램이므로 **AGPL 전염 회피가 가장 쉽다.**

#### ⑥ 콘텐츠·에셋 판매 (라이선스 무관)

프롬프트·템플릿·목소리·스타일은 코드가 아니므로 AGPL 대상이 아니다.

| 상품 | 가격 |
|---|---|
| 요약 스타일 팩 (학술/경영/뉴스/스레드) | $19 |
| 버티컬별 프롬프트 팩 | $29 |
| 프리미엄 TTS 목소리 팩 | $9 ~ 29 |
| 테마 팩 | $9 |
| Notion/Obsidian 템플릿 | $15 |

난이도·초기비용이 가장 낮아 첫 수익을 만들기 좋다.

#### ⑦ 교육 콘텐츠 — 가장 현실적인 출발점

이 코드베이스 자체가 교육 상품이 된다.

| 상품 | 가격대 |
|---|---|
| 온라인 강의 ("Tauri v2로 로컬 AI 앱 만들기") | 5만 ~ 15만원 |
| 유료 뉴스레터/멤버십 | 월 $10 |
| 보일러플레이트 판매 ("로컬 AI 데스크톱 앱 스타터킷") | $99 ~ 299 |
| 컨설팅 | 시간당 15만 ~ 30만원 |
| 유튜브/블로그 (광고 + 제휴) | — |

"Tauri + 로컬 LLM + 보안 설계"를 다루는 한국어 자료가 희소하다.
초기비용 0원, AGPL 무관(코드 설명은 자유), 개인 브랜딩 → 이후 ①②③으로 확장 가능.

#### ⑧ 하드웨어/어플라이언스 번들

미니PC(NPU 탑재)에 프리인스톨해 "AI 회의록 박스"로 판매.
150만 ~ 400만원, 전원만 연결하면 작동. 병원/법률사무소/소규모 기업 타깃.
AGPL은 하드웨어 판매를 금지하지 않는다(소스 접근만 제공).
재고·AS 부담이 있으나 마진이 좋다.

#### ⑨ API 비즈니스

```text
POST /v1/transcribe   { url }        → 전사문
POST /v1/summarize    { url, style } → 요약
POST /v1/chat         { id, q }      → 질의응답
```

과금: 오디오 분당 $0.006 / 요약당 $0.02.
경쟁사(AssemblyAI, Deepgram) 대비 "전사 + 요약 + 채팅 통합"으로 차별화.
안정성 요구가 높아 난이도가 있으나 확장성이 크다.

#### ⑩ 커뮤니티/데이터 플레이 (간접 수익)

- 공개 요약 라이브러리(유명 강연/컨퍼런스 아카이브) → SEO 트래픽 → 광고/제휴
- 뉴스레터("이번 주 AI 컨퍼런스 요약") → 스폰서십
- 이미 `share` tRPC 라우터와 `/share/[slug]` 페이지가 있어 공유 기반이 존재

### 12.2 종합 비교

| # | 아이디어 | 난이도 | 초기비용 | 수익규모 | 실행속도 | 라이선스 안전 |
|---|---|---|---|---|---|---|
| ⑦ | 교육 콘텐츠 | 낮음 | 0원 | 중 | 매우 빠름 | 안전 |
| ① | MCP 서버 | 낮음 | 낮음 | 중상 | 매우 빠름 | 설계 주의 |
| ③ | 버티컬 SaaS | 중간 | 중간 | 상 | 보통 | 클린룸 권장 |
| ② | B2B 온프레미스 | 중간(영업) | 낮음 | **최상** | 느림 | 안전 |
| ⑥ | 콘텐츠/에셋 | 매우 낮음 | 0원 | 낮음 | 매우 빠름 | 안전 |
| ⑤ | 커넥터 마켓 | 낮음 | 낮음 | 중 | 보통 | 안전 |
| ⑨ | API 비즈니스 | 높음 | 높음 | 상 | 느림 | 주의 |
| ④ | Managed 클라우드 | 높음 | 높음 | 중상 | 느림 | 주의 |
| ⑧ | 하드웨어 번들 | 높음 | 높음 | 상 | 느림 | 안전 |
| ⑩ | 커뮤니티/데이터 | 낮음 | 낮음 | 낮음 | 느림 | 안전 |

---

## 13. 실행 로드맵

### Phase 1 (1 ~ 2개월) — 비용 없는 것부터

- OpenBrief를 실제로 사용 (본인 영상 10개 요약)
- 분석 글/영상 시리즈 제작 (아이디어 ⑦) → 개인 브랜딩 시작
- 프롬프트/스타일 팩 제작 및 판매 (아이디어 ⑥) → 첫 수익
- 시장조사: 어느 버티컬이 가장 고통스러운지 사용자 인터뷰 10명

### Phase 2 (2 ~ 4개월) — 첫 제품

- MCP 서버 구현 (아이디어 ①) → 무료 공개로 인지도 확보
- 개발자 커뮤니티 + Product Hunt 등에 배포
- 무료 티어로 사용자 확보 후 유료 전환 실험
- 병행: 보일러플레이트 판매 (아이디어 ⑦)

### Phase 3 (4 ~ 8개월) — 본 제품

- Phase 1 조사에서 고통이 가장 컸던 버티컬 1개 선택 (아이디어 ③)
- React + Laravel(또는 Next.js)로 **클린룸 구현**
- 결제 연동(Stripe / Cashier) 후 유료 런칭
- 병행: B2B 온프레미스 문의 대응 (아이디어 ②) → 대형 단가 확보

### Phase 4 (8개월 이후) — 확장

- 커넥터 마켓플레이스 (⑤)
- 공개 API (⑨)
- 하드웨어 번들 파일럿 (⑧)

### 결론 요약

- 즉시 착수 권장: **⑦ 교육 콘텐츠 + ① MCP 서버** (초기비용 0, 라이선스 안전,
  브랜딩 효과, 이후 ②③으로 점프 가능)
- SaaS를 만들 때는 **클린룸 구현**으로 AGPL 리스크 회피
- 기술 선택보다 **"누구의 어떤 고통을 없애줄지"** 정의가 우선.
  Phase 1의 인터뷰 10명이 핵심이다.

---

## 14. 참고 자료

### 저장소

- 분석 대상: <https://github.com/bmshin94/openbrief>
- 원본(업스트림): <https://github.com/tantara/openbrief>
- 데모 영상: <https://youtu.be/OnS3EViayRo>

### 저장소 내 문서

- `README.md` — 제품 소개, 모델 지원 표, 셋업, 로드맵
- `AGENTS.md` — 아키텍처 규칙, 자기완결형 라이브러리 구조, 검증/커밋 규칙
- `docs/LOCAL_MODEL.md` — 로컬 모델 저장 규칙 및 디렉터리 레이아웃
- `docs/release.md` — 릴리즈 절차
- `docs/WINDOWS_SIGNING.md` — 윈도우 코드사인

### OpenBrief가 참고한 프로젝트 (README Acknowledgements)

- [yt-dlp](https://github.com/yt-dlp/yt-dlp) — 영상 다운로드
- [whisper.cpp](https://github.com/ggml-org/whisper.cpp),
  [transcribe-rs](https://github.com/cjpais/transcribe-rs) — 로컬 STT
- [FluidAudio](https://github.com/FluidInference/FluidAudio) — Apple 플랫폼 오디오 AI
- [Qwen3-ASR](https://github.com/QwenLM/Qwen3-ASR),
  [Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS)
- [Supertonic](https://github.com/supertone-inc/supertonic/) — Supertonic 3 TTS
- [tweakcn](https://tweakcn.com/) — shadcn 테마
- [Voicebox](https://github.com/jamiepine/voicebox),
  [Anarlog](https://github.com/fastrepl/anarlog) — 제품/구현 영감

### 라이선스

OpenBrief는 [GNU AGPL v3.0](../LICENSE)을 따른다.
2차 활용·수익화 시 본 문서 12.0절의 라이선스 전략을 반드시 검토할 것.
