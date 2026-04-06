# Claude Code 소스 코드 심층 분석

> **분석 기준일**: 2026년 4월 6일  
> **대상 버전**: claude-code-local 999.0.0-local  
> **소스 규모**: TypeScript/TSX 파일 1,894개, src/ 디렉토리 약 35MB  
> **참고**: 본 문서는 [wikidocs.net/338204](https://wikidocs.net/338204)의 분석 구조를 참고하여 실제 소스 코드를 정밀 분석한 결과물입니다.

---

## 목차

1. [Claude Code란 무엇인가](#1-claude-code란-무엇인가)
2. [전체 구조를 한눈에 보기](#2-전체-구조를-한눈에-보기)
3. [실행 모드](#3-실행-모드)
4. [시작과 초기화 과정](#4-시작과-초기화-과정)
5. [쿼리 루프 — 핵심 엔진](#5-쿼리-루프--핵심-엔진)
6. [QueryEngine — SDK용 래퍼](#6-queryengine--sdk용-래퍼)
7. [도구(Tool) 시스템](#7-도구tool-시스템)
8. [도구 실행 오케스트레이션](#8-도구-실행-오케스트레이션)
9. [명령어(Command) 시스템](#9-명령어command-시스템)
10. [태스크(Task) 시스템](#10-태스크task-시스템)
11. [상태 관리](#11-상태-관리)
12. [서비스 레이어](#12-서비스-레이어)
13. [권한(Permission) 시스템](#13-권한permission-시스템)
14. [훅(Hook) 시스템](#14-훅hook-시스템)
15. [스킬(Skill)과 플러그인(Plugin) 시스템](#15-스킬skill과-플러그인plugin-시스템)
16. [UI 레이어](#16-ui-레이어)
17. [브리지(Bridge) 시스템](#17-브리지bridge-시스템)
18. [원격(Remote) 세션 관리](#18-원격remote-세션-관리)
19. [코디네이터(Coordinator) 모드](#19-코디네이터coordinator-모드)
20. [메모리(Memory) 시스템](#20-메모리memory-시스템)
21. [타입 시스템과 상수](#21-타입-시스템과-상수)
22. [유틸리티 모듈](#22-유틸리티-모듈)
23. [핵심 설계 패턴](#23-핵심-설계-패턴)
24. [전체 데이터 흐름 요약](#24-전체-데이터-흐름-요약)

---

## 1. Claude Code란 무엇인가

Claude Code는 사용자의 컴퓨터에서 직접 동작하는 **AI 소프트웨어 엔지니어**다. 파일을 읽고, 코드를 수정하며, 셸 명령을 실행하고, 웹을 검색하는 등 개발자가 터미널에서 하는 거의 모든 작업을 수행할 수 있다.

### 기술 스택

| 구분 | 기술 | 버전 |
|------|------|------|
| **언어** | TypeScript (ES Modules) | - |
| **런타임** | Bun | - |
| **UI 프레임워크** | React + Ink (TUI) | React 19.2.4, Ink 6.8.0 |
| **레이아웃 엔진** | Yoga (Flexbox) | react-reconciler 0.33.0 |
| **상태 관리** | Zustand (DeepImmutable) | - |
| **API 클라이언트** | @anthropic-ai/sdk | 0.80.0 |
| **프로토콜** | Model Context Protocol (MCP) | @modelcontextprotocol/sdk 1.29.0 |
| **관측성** | OpenTelemetry | 트레이싱, 메트릭, 로그 |
| **CLI 파싱** | Commander.js | @commander-js/extra-typings 14.0.0 |
| **검색 엔진** | ripgrep (rg) | - |
| **스키마 검증** | Zod v4 | - |

### 핵심 의존성 구조

```
@anthropic-ai/sdk ─── Claude API 호출
@modelcontextprotocol/sdk ─── MCP 서버 통합
react + react-reconciler + ink ─── 터미널 UI 렌더링
@opentelemetry/* ─── 분산 추적, 메트릭, 로그
zod ─── 입력 스키마 검증
execa ─── 프로세스 실행
chokidar ─── 파일 변경 감시
lru-cache ─── 메모이제이션
```

---

## 2. 전체 구조를 한눈에 보기

Claude Code의 동작은 크게 **네 단계**로 나뉜다:

```
┌─────────────────────────────────────────────────────────────────┐
│                    사용자가 터미널에 입력                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 1: STARTUP & INITIALIZATION                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │ 인증     │ │ 모델선택 │ │ 설정로드 │ │ 컨텍스트 수집    │   │
│  │ (OAuth,  │ │ (Opus,   │ │ (JSON,   │ │ (시스템프롬프트, │   │
│  │ API Key, │ │ Sonnet,  │ │ MDM,     │ │  MCP, 플러그인)  │   │
│  │ Bedrock, │ │ Haiku)   │ │ 환경변수)│ │                  │   │
│  │ Vertex)  │ │          │ │          │ │                  │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 2: QUERY LOOP (AsyncGenerator)                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ 메시지 전처리 → API 스트리밍 → 에러 복구 → 도구 실행 루프 │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 3: TOOL EXECUTION                                        │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐   │
│  │ Bash │ │ Edit │ │ Read │ │ Grep │ │ Agent│ │ MCP Tool │   │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────────┘   │
│  입력검증 → 권한확인 → 실행 → 결과매핑 → 훅 실행              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Phase 4: DISPLAY & RENDERING                                   │
│  React/Ink TUI → 커스텀 Reconciler → Yoga 레이아웃 → ANSI 출력 │
└─────────────────────────────────────────────────────────────────┘
```

### 디렉토리 구조

```
claude-code-haha/
├── bin/claude-haha          # 실행 진입점 (bash 스크립트)
├── package.json             # 프로젝트 메타데이터
├── bunfig.toml              # Bun 런타임 설정
├── tsconfig.json            # TypeScript 설정
├── preload.ts               # Bun preload 스크립트
├── docs/                    # 아키텍처 다이어그램 (PNG 9장)
├── stubs/                   # 타입 스텁
└── src/                     # 소스 코드 (1,894개 파일)
    ├── entrypoints/         # 진입점 (cli.tsx, sdk/)
    ├── main.tsx             # 메인 앱 초기화 (4,690줄)
    ├── query.ts             # 쿼리 루프 핵심 엔진 (1,729줄)
    ├── QueryEngine.ts       # SDK 래퍼 (1,295줄)
    ├── Tool.ts              # 도구 인터페이스 (792줄)
    ├── tools.ts             # 도구 등록/조합 (389줄)
    ├── tools/               # 45+ 개별 도구 구현
    ├── screens/             # 화면 컴포넌트 (REPL, Doctor, Resume)
    ├── components/          # UI 컴포넌트
    ├── ink/                 # 커스텀 TUI 프레임워크
    ├── state/               # Zustand 상태 관리
    ├── services/            # 서비스 레이어 (22개 하위 디렉토리)
    ├── bridge/              # 브리지/원격 세션
    ├── coordinator/         # 멀티 에이전트 오케스트레이션
    ├── commands/            # 슬래시 커맨드 (80+ 종류)
    ├── tasks/               # 태스크 관리 (Agent, Shell, Remote)
    ├── hooks/               # 이벤트 훅
    ├── skills/              # 스킬 시스템
    ├── plugins/             # 플러그인 시스템
    ├── memdir/              # 메모리 디렉토리
    ├── utils/               # 유틸리티 (40+ 하위 디렉토리)
    ├── types/               # 타입 정의
    ├── constants/           # 상수
    ├── schemas/             # 스키마 정의
    ├── context/             # React 컨텍스트
    └── bootstrap/           # 부트스트랩 상태
```

---

## 3. 실행 모드

Claude Code는 하나의 바이너리에서 **여러 실행 모드**를 지원한다. `src/entrypoints/cli.tsx`(302줄)가 라우팅을 담당한다.

```
bin/claude-haha
    │
    ├─ 환경변수 CLAUDE_CODE_FORCE_RECOVERY_CLI=1 ──→ Recovery CLI (readline REPL)
    │
    └─ 기본 경로 ──→ src/entrypoints/cli.tsx
         │
         ├── --version / -v ──→ 버전 출력 후 종료
         ├── --dump-system-prompt ──→ 시스템 프롬프트 덤프
         ├── --claude-in-chrome-mcp ──→ Chrome MCP 서버 모드
         ├── --daemon-worker=<kind> ──→ 데몬 워커 서브프로세스
         ├── remote-control|rc|bridge ──→ 브리지 모드
         ├── daemon [subcommand] ──→ 데몬 수퍼바이저
         ├── ps|logs|attach|kill|--bg ──→ 백그라운드 세션 관리
         ├── new|list|reply ──→ 템플릿 작업 모드
         ├── environment-runner ──→ BYOC 러너 (게이트)
         ├── self-hosted-runner ──→ 셀프호스티드 러너 (게이트)
         │
         └── 기본값 ──→ src/main.tsx (전체 TUI 로드)
              │
              ├── REPL 모드 (기본): 대화형 터미널 UI
              ├── Doctor 모드: 진단/설정 화면
              ├── Resume 모드: 이전 세션 복원
              ├── Assistant 모드 (Kairos): 장시간 데몬
              └── Coordinator 모드: 멀티 에이전트 오케스트레이션
```

| 모드 | 진입 경로 | 특성 |
|------|----------|------|
| **REPL** | 기본 | Ink TUI, 대화형, 스트리밍 응답 |
| **Headless** | SDK/프로그래밍 | TUI 없음, API 직접 호출 |
| **Recovery CLI** | 환경변수 오버라이드 | readline 기반, 최소 의존성 |
| **Bridge** | `remote-control` 서브커맨드 | 로컬↔클라우드 연결 |
| **Coordinator** | 피처 게이트 | 리더+워커 멀티에이전트 |
| **Assistant (Kairos)** | 피처 게이트 | 장시간 백그라운드 데몬 |
| **Daemon** | `daemon` 서브커맨드 | 수퍼바이저+워커 프로세스 |

---

## 4. 시작과 초기화 과정

### 4.1 부트스트랩 시퀀스

`src/main.tsx`(4,690줄, 786KB)가 전체 초기화를 담당한다.

```
[1] 병렬 I/O 프리페치
    ├── startMdmRawRead()        ── MDM 설정 병렬 읽기 (macOS/Windows)
    ├── startKeychainPrefetch()  ── 키체인 병렬 읽기 (OAuth + API 키)
    └── profileCheckpoint()      ── 시작 프로파일러 마커

[2] 조건부 모듈 로딩 (Lazy Require)
    ├── Teammate 유틸리티
    ├── Coordinator 모드 (COORDINATOR_MODE 피처 게이트)
    └── Assistant 모드 (KAIROS 피처 게이트)

[3] 코어 모듈 로드 (순차)
    ├── 시스템/사용자 컨텍스트 빌더
    ├── 설정, 인증, 모델 설정
    ├── 플러그인/스킬 초기화
    ├── MCP 클라이언트
    ├── 도구 레지스트리
    ├── 정책 제한, GrowthBook 분석 게이트
    └── 부트스트랩 상태 관리

[4] 인증 확인
    ├── API Key (ANTHROPIC_API_KEY)
    ├── Auth Token (ANTHROPIC_AUTH_TOKEN)
    ├── OAuth (브라우저 기반 플로우)
    ├── Bedrock (AWS IAM)
    └── Vertex (Google 서비스 계정)

[5] 모델 해석
    └── 초기 모델 설정 → 폴백 체인 구성

[6] 렌더 트리 마운트
    └── Ink/React TUI → AppStateProvider → 화면 라우팅
```

### 4.2 글로벌 세션 상태

`src/bootstrap/state.ts`(1,300줄, 56KB)는 순환 의존 없는 **리프 모듈**로, 모든 전역 세션 상태를 관리한다.

```typescript
// 핵심 상태 필드 (요약)
type State = {
  originalCwd: string          // 최초 작업 디렉토리
  projectRoot: string          // 프로젝트 루트
  totalCostUSD: number         // 누적 API 비용
  totalAPIDuration: number     // 누적 API 소요 시간
  cwd: string                  // 현재 작업 디렉토리
  modelUsage: Record<string, ModelUsage>  // 모델별 토큰 사용량
  sessionId: SessionId         // 세션 고유 ID
  isInteractive: boolean       // 대화형 여부
  agentColorMap: Map<string, AgentColorName>  // 에이전트 색상 매핑
  lastAPIRequest: BetaMessageStreamParams | null  // 최근 API 요청
  sessionCronTasks: SessionCronTask[]  // 세션 크론 태스크
}
```

**설계 원칙**: Bootstrap은 utils와 types만 임포트하는 리프 모듈이므로 순환 의존이 발생하지 않는다.

---

## 5. 쿼리 루프 — 핵심 엔진

`src/query.ts`(1,729줄, 68KB)는 Claude Code의 **심장**이다. AsyncGenerator 패턴으로 구현되어, API 호출부터 도구 실행까지 전체 대화 루프를 관리한다.

### 5.1 함수 시그니처

```typescript
export async function* query(params: QueryParams): AsyncGenerator<
  | StreamEvent        // API 스트리밍 청크
  | RequestStartEvent  // 요청 시작 이벤트
  | Message            // 완성된 메시지
  | TombstoneMessage   // 삭제 표시 메시지
  | ToolUseSummaryMessage,  // 도구 사용 요약
  Terminal             // 루프 종료 시 반환값
>
```

### 5.2 단일 턴 실행 흐름

```
┌─────────────────────────────────────────────────────┐
│ [1] 메시지 전처리                                    │
│     ├── Tool Result Budget 적용 (결과 크기 제한)     │
│     ├── Snip Compaction (이력 압축, 피처게이트)      │
│     ├── Microcompact (도구 결과 캐싱 편집)           │
│     └── Context Collapse (피처게이트 요약 생성)      │
├─────────────────────────────────────────────────────┤
│ [2] API 스트리밍 호출                                │
│     ├── 시스템 프롬프트 조립                         │
│     ├── 메시지 정규화 (normalizeMessagesForAPI)      │
│     ├── 스트리밍 요청 → yield RequestStartEvent      │
│     └── 청크별 yield StreamEvent                    │
├─────────────────────────────────────────────────────┤
│ [3] 에러 Withhold & Recovery                        │
│     ├── max_output_tokens → 최대 3회 재시도          │
│     ├── prompt_too_long → 리액티브 컴팩션           │
│     └── API 에러 → 폴백 모델 전환                   │
├─────────────────────────────────────────────────────┤
│ [4] 도구 실행                                       │
│     ├── tool_use 블록 파싱                          │
│     ├── 권한 확인 (canUseTool)                      │
│     ├── StreamingToolExecutor 또는 순차 실행         │
│     └── tool_result 블록 누적                       │
├─────────────────────────────────────────────────────┤
│ [5] 후처리 및 계속 판단                              │
│     ├── stop_reason === 'end_turn' → Terminal 반환  │
│     ├── stop_reason === 'tool_use' → 루프 반복      │
│     ├── Reactive Compact 필요 → 컴팩션 후 반복      │
│     └── 예산 초과 → 예산 추적 갱신                  │
└─────────────────────────────────────────────────────┘
```

### 5.3 루프 내부 상태

```typescript
type State = {
  messages: Message[]                    // 대화 이력
  toolUseContext: ToolUseContext          // 도구 실행 컨텍스트
  autoCompactTracking: AutoCompactTrackingState | undefined  // 자동 컴팩션 추적
  maxOutputTokensRecoveryCount: number   // max_output 에러 복구 횟수
  hasAttemptedReactiveCompact: boolean   // 리액티브 컴팩션 시도 여부
  turnCount: number                      // 현재 턴 번호
  stopHookActive: boolean | undefined    // 정지 훅 활성화 여부
  transition: Continue | undefined       // 다음 상태 전이
}
```

### 5.4 피처 게이트에 의한 분기

| 피처 게이트 | 기능 | 설명 |
|------------|------|------|
| `REACTIVE_COMPACT` | 리액티브 컴팩션 | 컨텍스트 초과 시 인라인 압축 |
| `CONTEXT_COLLAPSE` | 컨텍스트 요약 | 긴 컨텍스트 점진적 요약 |
| `HISTORY_SNIP` | 이력 스닙 | 공격적 이력 트리밍 |
| `BG_SESSIONS` | 백그라운드 세션 | 태스크 요약 생성 |
| `TOKEN_BUDGET` | 토큰 예산 | 턴별 토큰 예산 추적 |
| `EXPERIMENTAL_SKILL_SEARCH` | 스킬 검색 | 프리페치 기반 스킬 발견 |

---

## 6. QueryEngine — SDK용 래퍼

`src/QueryEngine.ts`(1,295줄, 46KB)는 `query()` 제너레이터를 SDK/에이전트가 소비할 수 있도록 감싸는 래퍼다.

### 핵심 기능

```
query.ts (AsyncGenerator)
    │
    └─→ QueryEngine.ts (SDK Wrapper)
         ├── runQuery()          ─── 단일 쿼리 실행, Terminal 반환
         ├── streamQuery()       ─── 이벤트를 콜백으로 스트리밍
         ├── MessageSelector     ─── 메시지 필터링 UI
         ├── 세션 지속성          ─── 트랜스크립트 기록
         ├── 시그니처 블록 제거   ─── 모델 서명 스트립
         └── 첨부 파일 처리      ─── 파일 연산 결과 변환
```

---

## 7. 도구(Tool) 시스템

### 7.1 도구 인터페이스

`src/Tool.ts`(792줄, 29KB)에 정의된 도구 계약:

```typescript
export type Tool = {
  name: string                        // 도구 이름
  description: string                 // 모델에 전달할 설명
  inputSchema: ToolInputJSONSchema    // Zod 기반 입력 스키마

  // 실행
  call(input, context, canUseTool, parentMessage, onProgress): Promise<ToolResult>

  // 권한
  checkPermissions(input, context): Promise<PermissionResult>
  isReadOnly(): boolean               // 읽기 전용 여부
  isDestructive(): boolean            // 파괴적 작업 여부
  isConcurrencySafe(): boolean        // 병렬 실행 안전 여부

  // 렌더링
  renderToolUseMessage(): ReactElement
  renderToolResultMessage(): ReactElement

  // 선택적
  validateInput?(): ValidationResult
  prompt?(): SystemPrompt             // 도구별 시스템 프롬프트
}
```

### 7.2 전체 도구 목록

`src/tools.ts`(389줄)가 도구 등록을 관리한다. `getAllBaseTools()`에서 피처 게이트와 환경에 따라 도구를 조합한다.

| 도구 | 파일 경로 | 설명 |
|------|----------|------|
| **BashTool** | `src/tools/BashTool/` | 셸 명령 실행, 샌드박스 지원 |
| **FileReadTool** | `src/tools/FileReadTool/` | 파일 읽기, 이미지/PDF 지원 |
| **FileEditTool** | `src/tools/FileEditTool/` | 파일 인라인 편집 (문자열 치환) |
| **FileWriteTool** | `src/tools/FileWriteTool/` | 새 파일 작성 |
| **GrepTool** | `src/tools/GrepTool/` | ripgrep 기반 패턴 검색 |
| **GlobTool** | `src/tools/GlobTool/` | 파일 패턴 매칭 |
| **AgentTool** | `src/tools/AgentTool/` | 서브에이전트 생성/관리 |
| **SendMessageTool** | `src/tools/SendMessageTool/` | 에이전트 간 메시지 전송 |
| **SkillTool** | `src/tools/SkillTool/` | 스킬(프롬프트) 실행 |
| **WebFetchTool** | `src/tools/WebFetchTool/` | URL 콘텐츠 가져오기 |
| **WebSearchTool** | `src/tools/WebSearchTool/` | 웹 검색 |
| **AskUserQuestionTool** | `src/tools/AskUserQuestionTool/` | 사용자에게 질문 |
| **TodoWriteTool** | `src/tools/TodoWriteTool/` | 할일 목록 관리 |
| **NotebookEditTool** | `src/tools/NotebookEditTool/` | Jupyter 노트북 편집 |
| **EnterPlanModeTool** | `src/tools/EnterPlanModeTool/` | 계획 모드 진입 |
| **ExitPlanModeTool** | `src/tools/ExitPlanModeTool/` | 계획 모드 종료 |
| **EnterWorktreeTool** | `src/tools/EnterWorktreeTool/` | Git 워크트리 진입 |
| **ExitWorktreeTool** | `src/tools/ExitWorktreeTool/` | Git 워크트리 종료 |
| **TaskCreateTool** | `src/tools/TaskCreateTool/` | 태스크 생성 |
| **TaskGetTool** | `src/tools/TaskGetTool/` | 태스크 조회 |
| **TaskListTool** | `src/tools/TaskListTool/` | 태스크 목록 |
| **TaskStopTool** | `src/tools/TaskStopTool/` | 태스크 중지 |
| **TaskUpdateTool** | `src/tools/TaskUpdateTool/` | 태스크 갱신 |
| **TaskOutputTool** | `src/tools/TaskOutputTool/` | 태스크 출력 조회 |
| **ToolSearchTool** | `src/tools/ToolSearchTool/` | 도구 검색 (Deferred) |
| **LSPTool** | `src/tools/LSPTool/` | Language Server Protocol |
| **MCPTool** | `src/tools/MCPTool/` | MCP 서버 도구 래퍼 |
| **TeamCreateTool** | `src/tools/TeamCreateTool/` | 팀 생성 |
| **TeamDeleteTool** | `src/tools/TeamDeleteTool/` | 팀 삭제 |
| **BriefTool** | `src/tools/BriefTool/` | 간략 응답 |
| **TungstenTool** | `src/tools/TungstenTool/` | 내부 도구 |
| **RemoteTriggerTool** | `src/tools/RemoteTriggerTool/` | 원격 에이전트 트리거 |
| **ScheduleCronTool** | `src/tools/ScheduleCronTool/` | 크론 스케줄링 |
| **ConfigTool** | `src/tools/ConfigTool/` | 설정 변경 |
| **ListMcpResourcesTool** | `src/tools/ListMcpResourcesTool/` | MCP 리소스 목록 |
| **ReadMcpResourceTool** | `src/tools/ReadMcpResourceTool/` | MCP 리소스 읽기 |
| **PowerShellTool** | `src/tools/PowerShellTool/` | PowerShell 실행 (Windows) |
| **REPLTool** | `src/tools/REPLTool/` | REPL 실행 (ant-only) |
| **SleepTool** | `src/tools/SleepTool/` | 대기 (Kairos/Proactive) |
| **SyntheticOutputTool** | `src/tools/SyntheticOutputTool/` | 테스트/녹화용 |
| **WorkflowTool** | `src/tools/WorkflowTool/` | 워크플로 실행 |

### 7.3 도구 등록 파이프라인

```
getAllBaseTools()
    │  정적 도구 + 피처게이트 조건부 도구
    ▼
getTools(permissionContext)
    │  deny 규칙 필터링, SIMPLE 모드 처리
    ▼
assembleToolPool(permissionContext, mcpTools)
    │  내장 도구 + MCP 도구 병합, 이름 충돌 시 내장 우선
    ▼
mergeAndFilterTools()
    │  React 훅, REPL 내 동적 도구 조합
    ▼
최종 도구 목록 → query()에 전달
```

### 7.4 피처 게이트에 의한 조건부 로딩

```typescript
// Dead Code Elimination: 빌드 시점에 피처가 꺼져 있으면 코드 자체가 제거됨
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [CronCreateTool, CronDeleteTool, CronListTool]
  : []

const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

---

## 8. 도구 실행 오케스트레이션

### 8.1 10단계 실행 파이프라인

API가 `tool_use` 블록을 반환하면, 다음 10단계 파이프라인을 거친다:

```
┌────┐  ┌────────────────────────────────────────────────┐
│ 1  │  │ 도구 조회: findToolByName(tools, toolName)     │
├────┤  │ 별칭(alias) 해석 포함                          │
│ 2  │  ├────────────────────────────────────────────────┤
│    │  │ Abort 시그널 확인                              │
├────┤  ├────────────────────────────────────────────────┤
│ 3  │  │ 입력 검증: tool.inputSchema.parse(input)       │
│    │  │ Zod v4 스키마 기반 검증                        │
├────┤  ├────────────────────────────────────────────────┤
│ 4  │  │ PreToolUse 훅 실행                             │
│    │  │ 셸 커맨드/HTTP/에이전트/프롬프트 형태          │
├────┤  ├────────────────────────────────────────────────┤
│ 5  │  │ 권한 확인: hasPermissionsToUseTool()            │
│    │  │ 규칙 매칭 → 모드 판단 → AI 분류기(선택적)     │
├────┤  ├────────────────────────────────────────────────┤
│ 6  │  │ 도구 실행: tool.call(input, context)            │
│    │  │ onProgress 콜백으로 진행 상황 스트리밍         │
├────┤  ├────────────────────────────────────────────────┤
│ 7  │  │ 결과 매핑: mapToolResultToToolResultBlockParam  │
│    │  │ API 형식으로 변환                              │
├────┤  ├────────────────────────────────────────────────┤
│ 8  │  │ 대형 결과 영속화: maxResultSizeChars 초과 시   │
│    │  │ 디스크에 저장, 모델에는 미리보기 + 경로 전달   │
├────┤  ├────────────────────────────────────────────────┤
│ 9  │  │ PostToolUse 훅 실행                            │
│    │  │ 성공/실패에 따라 다른 훅 트리거               │
├────┤  ├────────────────────────────────────────────────┤
│ 10 │  │ 텔레메트리 로깅                                │
│    │  │ OpenTelemetry 스팬 기록                        │
└────┘  └────────────────────────────────────────────────┘
```

### 8.2 동시성 파티셔닝

도구는 `isConcurrencySafe()` 플래그에 따라 병렬/순차 실행이 결정된다:

```
입력:  [Read] [Grep] [Glob] [Edit] [Read] [Read] [Bash]

Batch 1: [Read, Grep, Glob]  ──→ 병렬 실행 (최대 10개)
Batch 2: [Edit]              ──→ 단독 순차 실행
Batch 3: [Read, Read]        ──→ 병렬 실행
Batch 4: [Bash]              ──→ 단독 순차 실행
```

**규칙**: 안전한 도구(Read, Grep, Glob)는 병렬로, 위험한 도구(Edit, Bash, Write)는 순차로 실행한다. `StreamingToolExecutor`가 이 배치 분할을 담당한다.

---

## 9. 명령어(Command) 시스템

`src/commands/` 디렉토리에 80개 이상의 슬래시 커맨드가 구현되어 있다.

### 주요 명령어 카테고리

| 카테고리 | 명령어 | 설명 |
|---------|--------|------|
| **세션** | /clear, /resume, /exit, /session, /compact | 세션 관리 |
| **설정** | /config, /model, /theme, /permissions, /env | 설정 변경 |
| **정보** | /help, /cost, /usage, /stats, /doctor, /status | 상태 확인 |
| **도구** | /mcp, /plugin, /skills, /hooks | 확장 기능 관리 |
| **작업** | /tasks, /plan, /diff, /review, /share, /export | 작업 관리 |
| **에이전트** | /agents, /context | 에이전트/컨텍스트 관리 |
| **메모리** | /memory | 영속 메모리 관리 |
| **UI** | /fast, /effort, /vim, /voice, /color | UI/UX 설정 |
| **인증** | /login, /logout, /oauth-refresh | 인증 관리 |

### 명령어 인터페이스

```typescript
type Command = {
  name: string             // 명령어 이름 (예: "help")
  description: string      // 설명
  pattern: string          // 정규식 매칭 패턴
  handler: (args: string) => Promise<void>  // 실행 핸들러
}
```

### 명령어 생명주기

```
사용자 입력 ("/help")
    ↓
패턴 매칭 (regex)
    ↓
실행 큐에 추가
    ↓
생명주기 이벤트 통지 (notifyCommandLifecycle)
    ↓
핸들러 실행 (AppState 변이 포함)
```

---

## 10. 태스크(Task) 시스템

`src/tasks/` 디렉토리에서 백그라운드 작업을 관리한다.

### 태스크 타입

```
┌──────────────────────────────────────────────┐
│                  Task System                  │
├──────────────┬───────────────────────────────┤
│ LocalAgent   │ 서브에이전트 생성, 격리된     │
│ Task         │ QueryEngine 인스턴스           │
├──────────────┼───────────────────────────────┤
│ LocalShell   │ 장시간 셸 명령 실행,          │
│ Task         │ Kill 시그널 처리              │
├──────────────┼───────────────────────────────┤
│ RemoteAgent  │ 브리지 기반 원격 실행,        │
│ Task         │ 이벤트 스트리밍               │
├──────────────┼───────────────────────────────┤
│ InProcess    │ 인프로세스 팀메이트 실행      │
│ Teammate     │                               │
├──────────────┼───────────────────────────────┤
│ DreamTask    │ 백그라운드 사색 태스크        │
│              │                               │
└──────────────┴───────────────────────────────┘
```

### 태스크 상태 모델

```typescript
type TaskState = {
  type: 'local_agent' | 'local_shell' | 'remote_agent' | 'dream'
  taskId: string                    // 고유 ID (접두사 's')
  status: 'running' | 'done' | 'background'
  abortController: AbortController  // 취소 제어
  transcript: Message[]             // 대화 기록
  selectedAgent?: AgentDefinition   // 선택된 에이전트 정의
}
```

---

## 11. 상태 관리

### 11.1 Zustand 스토어

`src/state/AppStateStore.ts`(569줄)와 `src/state/AppState.tsx`(199줄)가 전체 애플리케이션 상태를 관리한다.

```typescript
type AppState = {
  // 핵심 설정
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  
  // UI 상태
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  selectedIPAgentIndex: number
  footerSelection: FooterItem | null
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  
  // 권한
  toolPermissionContext: ToolPermissionContext
  
  // 에이전트
  agent: string | undefined
  kairosEnabled: boolean
  
  // 원격/브리지
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  replBridgeEnabled: boolean
  
  // ... 20개 이상의 추가 필드
}
```

### 11.2 상태 접근 패턴

```
AppStateStore (Zustand 싱글톤)
    │
    ├── AppStateProvider (React Context)
    │       └── useAppState(selector) ─── 메모이즈드 선택자
    │
    ├── store.setState({...}) ─── 직접 상태 갱신
    │
    └── 외부 리스너
          ├── settingsChangeDetector ─── 설정 파일 변경 감시
          └── skillChangeDetector ─── 스킬 파일 변경 감시
```

**설계 원칙**: `DeepImmutable<T>` 타입으로 우발적 변이를 방지하고, 선택자 메모이제이션으로 불필요한 리렌더링을 차단한다.

---

## 12. 서비스 레이어

`src/services/` 디렉토리(22개 하위 디렉토리, 290+ 파일)가 핵심 비즈니스 로직을 캡슐화한다.

### 서비스 맵

```
services/
├── api/                ─── Claude API 통신
│   ├── claude.ts           Anthropic SDK 스트리밍 래퍼
│   ├── withRetry.ts        지수 백오프 재시도
│   ├── errors.ts           에러 유형 정의
│   └── filesApi.ts         파일 업로드/다운로드
│
├── analytics/          ─── 분석/텔레메트리
│   └── GrowthBook 통합, 세션 추적, 이벤트 로깅
│
├── mcp/                ─── Model Context Protocol
│   ├── client.ts           MCP 클라이언트 (119KB)
│   ├── config.ts           서버 설정 (51KB)
│   ├── auth.ts             OAuth 인증 (88KB)
│   └── types.ts            타입 정의
│
├── compact/            ─── 컨텍스트 압축
│   ├── autoCompact.ts      자동 컴팩션 임계값 관리
│   ├── compact.ts          메시지 이력 요약
│   └── reactiveCompact.ts  리액티브 인라인 압축
│
├── contextCollapse/    ─── 긴 컨텍스트 요약
│
├── SessionMemory/      ─── 세션 메모리
│   ├── sessionMemory.ts    자동 메모리 추출
│   └── sessionMemoryUtils.ts  임계값/상태 추적
│
├── extractMemories/    ─── 영속 메모리 추출
│
├── oauth/              ─── OAuth 인증
│   ├── auth-code-listener.ts  리디렉트 리스너
│   └── auth.ts               토큰 관리
│
├── tools/              ─── 도구 실행
│   └── StreamingToolExecutor  동시 도구 실행기
│
├── toolUseSummary/     ─── 도구 사용 요약 생성
│
├── lsp/                ─── Language Server Protocol
│
├── plugins/            ─── 플러그인 관리
│
├── policyLimits/       ─── 정책 제한
│
├── remoteManagedSettings/ ─── 원격 관리 설정
│
├── settingsSync/       ─── 설정 동기화
│
├── teamMemorySync/     ─── 팀 메모리 동기화
│
├── tips/               ─── 팁 시스템
│
├── AgentSummary/       ─── 에이전트 요약
│
├── MagicDocs/          ─── 문서 자동 생성
│
├── PromptSuggestion/   ─── 프롬프트 제안
│
└── autoDream/          ─── 자동 사색
```

### API 통신 상세

```
사용자 메시지
    ↓
normalizeMessagesForAPI()   ─── 메시지 정규화
    ↓
prependUserContext()        ─── 사용자 컨텍스트 삽입
appendSystemContext()       ─── 시스템 컨텍스트 삽입
    ↓
Anthropic SDK streaming     ─── Beta API 호출
    ↓
withRetry()                 ─── 재시도 래퍼
    ├── 지수 백오프 (연결 실패)
    ├── prompt_too_long → 컴팩션 트리거
    └── 할당량 초과 → 폴백 모델 전환
    ↓
StreamEvent yield           ─── 청크별 이벤트 발생
```

---

## 13. 권한(Permission) 시스템

### 13.1 권한 모드

| 모드 | 동작 | 사용 사례 |
|------|------|----------|
| **default** | 위험한 도구 사용 시 사용자에게 묻기 | 일반 사용 |
| **bypassPermissions** | 모든 도구 무조건 허용 | 신뢰 환경 |
| **acceptEdits** | 작업 디렉토리 내 파일 편집 자동 허용 | 편집 중심 작업 |
| **dontAsk** | 허용되지 않은 도구는 자동 거부 | 비대화형 환경 |
| **auto** | AI 분류기로 판단 | 자동화 환경 |
| **plan** | 읽기 전용 (auto 모드 활성화 가능) | 계획 수립 단계 |

### 13.2 권한 판정 파이프라인

`src/utils/permissions/permissions.ts`가 핵심 로직을 담당한다.

```
┌──────────────────────────────────────────────────────────┐
│ [1] 입력 검증: tool.validateInput()                      │
│     → 유효하지 않으면 즉시 거부                          │
├──────────────────────────────────────────────────────────┤
│ [2] 도구 자체 권한 확인: tool.checkPermissions()          │
│     → 도구별 커스텀 로직 (예: 파일 경로 검증)           │
├──────────────────────────────────────────────────────────┤
│ [3] PreToolUse 훅 실행                                   │
│     → 훅이 차단/수정 가능                               │
├──────────────────────────────────────────────────────────┤
│ [4] 규칙 기반 매칭                                       │
│     ├── 도구 전체 거부 규칙 → DENY                      │
│     ├── 도구 전체 허용 규칙 → ALLOW                     │
│     ├── 콘텐츠별 거부 규칙 → DENY                       │
│     ├── 콘텐츠별 허용 규칙 → ALLOW                      │
│     └── 매칭 없음 → ASK                                │
├──────────────────────────────────────────────────────────┤
│ [5] 모드별 변환                                          │
│     ├── dontAsk: ASK → DENY                             │
│     ├── auto: ASK → AI 분류기 호출                      │
│     ├── acceptEdits: 작업 디렉토리 내 → ALLOW           │
│     └── default: ASK 유지 (사용자 프롬프트)             │
└──────────────────────────────────────────────────────────┘
```

### 13.3 권한 규칙 형식

```
"ToolName"              → 도구 전체에 적용
"ToolName(content)"     → 특정 콘텐츠 패턴에 적용
"Bash(npm install)"     → npm install 명령만 허용/거부
"FileEdit(/src/*.ts)"   → 특정 경로 패턴에 적용
"mcp__serverName__*"    → MCP 서버의 모든 도구에 적용
```

### 13.4 규칙 소스 우선순위

```
[1] flagSettings      ─── 피처 플래그 (최우선)
[2] policySettings    ─── 관리자 정책 (강제)
[3] localSettings     ─── 로컬 시스템 설정
[4] projectSettings   ─── 프로젝트별 .claude/settings.json
[5] userSettings      ─── 사용자 ~/.claude/settings.json
[6] cliArg            ─── 커맨드라인 인수
[7] command/session   ─── 런타임/세션 규칙
```

### 13.5 AI 분류기 (Auto 모드)

`TRANSCRIPT_CLASSIFIER` 피처 게이트가 활성화되면 AI 분류기를 사용한다:

```
ASK 결정 도달
    ↓
acceptEdits 모드에서 허용될까? → Yes → ALLOW (빠른 경로)
    ↓ No
안전 도구 허용 목록에 있나? → Yes → ALLOW (빠른 경로)
    ↓ No
classifyYoloAction() 호출 ─── AI 분류기 API
    ├── 2단계 평가 (빠른 + 사고)
    ├── 토큰 사용량 텔레메트리 추적
    ├── 연속 거부 횟수 추적
    └── 임계값 초과 시 사용자 프롬프트로 폴백
```

### 13.6 판정 추적 (감사 추적)

모든 권한 결정에는 `PermissionDecisionReason`이 첨부된다:

```typescript
type PermissionDecisionReason =
  | { type: 'rule', rule: PermissionRule }
  | { type: 'mode', mode: PermissionMode }
  | { type: 'hook', hookName, hookSource }
  | { type: 'classifier', classifier, reason }
  | { type: 'workingDir', reason }
  | { type: 'safetyCheck', reason, classifierApprovable }
  | { type: 'sandboxOverride' }
  | { type: 'permissionPromptTool' }
  | { type: 'other', reason }
```

---

## 14. 훅(Hook) 시스템

`src/utils/hooks.ts`가 이벤트 기반 훅 실행 엔진을 제공한다.

### 훅 이벤트 유형

```
┌────────────────────┬──────────────────────────────────────┐
│ 이벤트              │ 발생 시점                            │
├────────────────────┼──────────────────────────────────────┤
│ session_start      │ 세션 시작 시                         │
│ session_end        │ 세션 종료 시                         │
│ setup              │ 첫 쿼리 전                          │
├────────────────────┼──────────────────────────────────────┤
│ pre_tool_use       │ 도구 실행 전 (차단/수정 가능)       │
│ post_tool_use      │ 도구 성공 후                        │
│ post_tool_use_failure │ 도구 실패 후                     │
├────────────────────┼──────────────────────────────────────┤
│ pre_compact        │ 컨텍스트 압축 전                    │
│ post_compact       │ 압축 후                              │
├────────────────────┼──────────────────────────────────────┤
│ permission_denied  │ 권한 거부 시                         │
│ permission_request │ 커스텀 권한 로직                     │
├────────────────────┼──────────────────────────────────────┤
│ notification       │ 시스템 알림                          │
│ prompt_request     │ 커스텀 승인 다이얼로그              │
└────────────────────┴──────────────────────────────────────┘
```

### 훅 실행 방식

```
훅 트리거
    ↓
매칭 규칙 확인 (예: "Bash(git *)")
    ↓
실행 형태 선택
    ├── 셸 서브프로세스 (bash, zsh, PowerShell)
    ├── HTTP 요청 (POST + JSON 본문)
    ├── 에이전트 서브에이전트 (재귀적 실행)
    ├── 프롬프트 요청 (stdin 통한 사용자 입력)
    └── 등록된 JS 함수 (세션 훅)
    ↓
JSON 출력 파싱 및 검증
    ↓
OpenTelemetry 스팬 기록
```

### 훅 설정 위치

```json
// ~/.claude/settings.json
{
  "hooks": {
    "pre_tool_use": [
      {
        "if": "Bash(git *)",
        "command": "echo '{\"allow\": true}'"
      }
    ],
    "post_tool_use": [
      {
        "command": "notify-send 'Tool completed'"
      }
    ]
  }
}
```

---

## 15. 스킬(Skill)과 플러그인(Plugin) 시스템

### 15.1 스킬 시스템

`src/skills/`와 `src/tools/SkillTool/SkillTool.ts`가 스킬 시스템을 구현한다.

**스킬이란?** YAML 프론트매터가 붙은 프롬프트 파일로, 포크된 서브에이전트 컨텍스트에서 실행된다.

```
스킬 발견
    ├── 로컬 파일시스템 스캔 (src/skills/bundled/)
    └── MCP 서버 스킬 (MCP_SKILLS 피처 게이트)
    ↓
getAllCommands()  ─── 로컬 + MCP 통합 목록
    ↓
SkillTool.call()
    ├── 포크된 서브에이전트에서 실행
    ├── 격리된 토큰 예산
    ├── 모델 오버라이드 가능
    └── 사용량 분석 추적
```

### 15.2 플러그인 시스템

`src/plugins/`가 플러그인 생명주기를 관리한다.

```
plugins/
├── bundled/        ─── 내장 플러그인
├── loader.ts       ─── 플러그인 로더
├── installer.ts    ─── 설치 관리자
└── manager.ts      ─── 생명주기 관리
```

**플러그인 기능**:
- 도구 확장 (커스텀 도구 등록)
- 설정 확장
- 커맨드 확장
- 훅 확장

---

## 16. UI 레이어

### 16.1 커스텀 TUI 프레임워크

`src/ink/` 디렉토리에 Ink 기반 커스텀 TUI 프레임워크가 구현되어 있다.

```
React 컴포넌트
    ↓
reconciler.ts (React Fiber 커스텀 조정기)
    ↓
DOMElement 트리 (dom.ts)
    ↓
Yoga 레이아웃 엔진 (layout/engine.ts)
    ↓
render-node-to-output.ts (DOM → 렌더 연산)
    ↓
Output.ts (연산 배치, 텍스트 토큰화)
    ↓
screen.ts (셀 기반 스크린 버퍼)
    ↓
render-to-screen.ts (연산 → 스크린)
    ↓
ANSI 이스케이프 시퀀스 출력
```

### 16.2 렌더링 최적화

| 기법 | 구현 위치 | 설명 |
|------|----------|------|
| **더블 버퍼링** | renderer.ts | frontFrame/backFrame 교차로 깜빡임 방지 |
| **오브젝트 풀링** | screen.ts | 문자열, 스타일, 하이퍼링크 재사용 |
| **더티 트래킹** | render-to-screen.ts | 변경된 셀만 업데이트 |
| **프레임 스로틀링** | renderer.ts | 렌더링 빈도 제어 |
| **그래핌 클러스터링** | output.ts | 터미널 너비 캐싱 |

### 16.3 터미널 I/O 프리미티브

```
src/ink/termio/
├── parser.ts   ─── ANSI 이스케이프 시퀀스 파서
├── sgr.ts      ─── Select Graphic Rendition (색상/스타일)
├── csi.ts      ─── Control Sequence Introducer
├── dec.ts      ─── DEC Private Mode
└── osc.ts      ─── Operating System Command
```

### 16.4 화면 레이아웃

```
┌─────────────────────────────────────────────┐
│ 헤더 (모델명, 세션 정보)                     │
├─────────────────────────────────────────────┤
│                                             │
│ 가상화된 메시지 목록                         │
│ (스트리밍 응답, 도구 결과, 코드 블록)        │
│                                             │
├─────────────────────────────────────────────┤
│ 태스크 패널 (활성 에이전트, 진행 상황)       │
├─────────────────────────────────────────────┤
│ 입력 프롬프트                                │
├─────────────────────────────────────────────┤
│ 상태 바 (비용, 모델, 브리지 상태)            │
└─────────────────────────────────────────────┘
```

### 16.5 주요 화면 컴포넌트

| 컴포넌트 | 파일 | 줄 수 | 설명 |
|---------|------|-------|------|
| **REPL** | `src/screens/REPL.tsx` | 5,005줄 | 메인 대화형 루프 |
| **Doctor** | `src/screens/Doctor.tsx` | 574줄 | 진단/문제해결 |
| **Resume** | `src/screens/ResumeConversation.tsx` | 398줄 | 세션 복원 |

### 16.6 UI 컴포넌트 구조

```
src/components/
├── PromptInput/       ─── 입력 프롬프트
├── messages/          ─── 메시지 렌더링
├── permissions/       ─── 권한 다이얼로그
├── tasks/             ─── 태스크 UI
├── agents/            ─── 에이전트 UI
├── teams/             ─── 팀 UI
├── diff/              ─── 디프 뷰어
├── StructuredDiff/    ─── 구조적 디프
├── HighlightedCode/   ─── 코드 하이라이팅
├── Spinner/           ─── 로딩 스피너
├── mcp/               ─── MCP 연결 관리
├── memory/            ─── 메모리 UI
├── sandbox/           ─── 샌드박스 UI
├── skills/            ─── 스킬 UI
├── design-system/     ─── 디자인 시스템
├── grove/             ─── 그로브 컴포넌트
├── ui/                ─── 기본 UI 요소
├── wizard/            ─── 위저드 플로우
├── hooks/             ─── 커스텀 React 훅
├── Settings/          ─── 설정 UI
├── TrustDialog/       ─── 신뢰 대화상자
├── LogoV2/            ─── 로고 렌더링
├── HelpV2/            ─── 도움말 화면
├── FeedbackSurvey/    ─── 피드백 설문
├── Passes/            ─── 패스 정보
└── shell/             ─── 셸 UI
```

---

## 17. 브리지(Bridge) 시스템

`src/bridge/`(32개 파일, 540KB)가 로컬 머신과 claude.ai를 연결하는 브리지 시스템을 구현한다.

### 아키텍처

```
┌──────────────┐     WebSocket      ┌──────────────────┐
│              │ ◄────────────────► │                  │
│  로컬 머신   │                    │  CCR Backend     │
│  (Bridge)    │     HTTP API       │  (Cloud Code     │
│              │ ◄────────────────► │   Remote)        │
└──────────────┘                    └──────────────────┘

bridgeMain.ts (115KB) ─── 코어 브리지 코디네이터
replBridge.ts (100KB) ─── REPL 브리지 트랜스포트
```

### 핵심 파일

| 파일 | 설명 |
|------|------|
| `bridgeMain.ts` | 메인 브리지 루프, 멀티세션 관리 |
| `bridgeApi.ts` | CCR HTTP 클라이언트, 토큰 리프레시 |
| `bridgeConfig.ts` | 브리지 설정 |
| `bridgeUI.ts` | 상태 표시 로거 |
| `bridgeMessaging.ts` | 메시지 직렬화 |
| `bridgePermissionCallbacks.ts` | 도구 권한 콜백 |
| `sessionRunner.ts` | 세션 스포닝 |
| `jwtUtils.ts` | JWT 토큰 관리 |
| `trustedDevice.ts` | 디바이스 신뢰 토큰 |

### 브리지 기능

- **멀티세션**: 최대 32개 동시 세션 (`tengu_ccr_bridge_multi_session` 게이트)
- **인증**: JWT + 디바이스 신뢰 토큰 (OAuth v1, JWT v2)
- **재연결**: 지수 백오프 재연결 로직
- **수면/깨기 감지**: 시스템 수면/깨기 상태 감지 후 적절한 복구

---

## 18. 원격(Remote) 세션 관리

`src/remote/` 디렉토리에서 원격 세션 인프라를 관리한다.

```
로컬 Claude Code
    │
    ├── 브리지 연결 수립
    │       ├── WebSocket 핸드셰이크
    │       ├── 토큰 교환 (OAuth → JWT)
    │       └── 디바이스 신뢰 검증
    │
    ├── 세션 생성/복원
    │       ├── 작업 디렉토리 설정
    │       ├── 세션 상태 동기화
    │       └── 폴링 루프 시작
    │
    └── 메시지 라우팅
            ├── 인바운드: 클라우드 → 로컬 (명령, 도구 실행)
            ├── 아웃바운드: 로컬 → 클라우드 (결과, 이벤트)
            └── 양방향 스트리밍
```

---

## 19. 코디네이터(Coordinator) 모드

`src/coordinator/coordinatorMode.ts`(19KB)가 멀티에이전트 오케스트레이션을 구현한다.

### 아키텍처

```
┌────────────────────────────────────────────┐
│            Coordinator (리더)               │
│  ┌──────────────────────────────────────┐  │
│  │ 전략적 위임만 수행                    │  │
│  │ 직접 코드 편집 금지                   │  │
│  │ 결과 취합 및 사용자 응답              │  │
│  └───────────┬──────────────────────────┘  │
│              │ AgentTool / SendMessageTool  │
│     ┌────────┴────────┐                    │
│     ▼                 ▼                    │
│  ┌──────┐         ┌──────┐                 │
│  │Worker│         │Worker│  격리된          │
│  │  #1  │         │  #2  │  QueryEngine    │
│  └──────┘         └──────┘  인스턴스        │
└────────────────────────────────────────────┘
```

### 실행 단계

```
[1] Research (병렬)
    ├── 워커들이 독립적으로 코드베이스 탐색
    └── 각자의 발견사항을 리더에게 보고

[2] Synthesis (리더 검토 필수)
    ├── 리더가 모든 연구 결과를 취합
    └── 구현 전략 수립

[3] Implement (순차)
    ├── 워커들이 순차적으로 구현 작업 수행
    └── 충돌 방지를 위한 직렬화

[4] Verify (병렬)
    ├── 워커들이 독립적으로 검증
    └── 리더가 최종 결과 취합
```

### 코디네이터 도구

| 도구 | 용도 |
|------|------|
| AgentTool | 새 워커 에이전트 생성 |
| SendMessageTool | 기존 워커에게 메시지 전송 |
| TaskStopTool | 워커 종료 |

---

## 20. 메모리(Memory) 시스템

### 20.1 영속 메모리 구조

```
~/.claude/projects/{project-slug}/memory/
├── MEMORY.md                ─── 메모리 인덱스 (200줄 제한)
├── user_role.md             ─── 사용자 역할/전문성
├── feedback_testing.md      ─── 작업 방식 피드백
├── project_freeze.md        ─── 프로젝트 상태/일정
└── reference_linear.md      ─── 외부 시스템 참조
```

### 20.2 메모리 타입

| 타입 | 설명 | 저장 시점 |
|------|------|----------|
| **user** | 사용자의 역할, 목표, 지식 | 사용자 정보 파악 시 |
| **feedback** | 작업 방식 교정/확인 | 사용자가 수정하거나 확인할 때 |
| **project** | 진행 중인 작업, 일정, 이슈 | 프로젝트 상태 변경 시 |
| **reference** | 외부 시스템 포인터 | 외부 리소스 발견 시 |

### 20.3 메모리 파일 형식

```markdown
---
name: 사용자 역할
description: 사용자는 시니어 백엔드 엔지니어, Go 전문
type: user
---

사용자는 10년 경력의 Go 백엔드 엔지니어로, 이 프로젝트의 React 프론트엔드는 처음이다.
프론트엔드 설명 시 백엔드 유사 개념으로 프레이밍할 것.
```

### 20.4 세션 메모리

`src/services/SessionMemory/`가 세션 내 자동 메모리 추출을 담당한다.

```
세션 진행 중
    ↓
임계값 확인:
    ├── minimumMessageTokensToInit: 10,000 토큰
    ├── minimumTokensBetweenUpdate: 5,000 토큰
    └── toolCallsBetweenUpdates: 3 도구 호출
    ↓
임계값 초과 시
    ↓
포크된 서브프로세스에서 메모리 추출 (비차단)
    ↓
.claude/MEMORY.md에 기록
    ↓
세션 종료 시 추출 완료 대기
```

---

## 21. 타입 시스템과 상수

### 21.1 타입 디렉토리

```
src/types/
├── generated/        ─── 자동 생성된 타입
├── message.ts        ─── 메시지 타입 (UserMessage, AssistantMessage, etc.)
├── permissions.ts    ─── 권한 관련 타입
└── tools.ts          ─── 도구 진행 상황 타입
```

### 21.2 핵심 메시지 타입

```typescript
type Message =
  | UserMessage           // 사용자 입력
  | AssistantMessage      // AI 응답
  | SystemMessage         // 시스템 메시지
  | AttachmentMessage     // 첨부 파일 (메모리, 컨텍스트)
  | ProgressMessage       // 진행 상황
  | SystemLocalCommandMessage  // 로컬 명령 실행

type StreamEvent = {
  type: 'stream'
  chunk: ContentBlock     // 스트리밍 청크
}

type RequestStartEvent = {
  type: 'request_start'
  requestId: string
}
```

### 21.3 상수 디렉토리

```
src/constants/
└── querySource.ts    ─── 쿼리 소스 상수 (REPL, SDK, Bridge, etc.)
```

---

## 22. 유틸리티 모듈

`src/utils/` 디렉토리(40개 이상 하위 디렉토리)에 광범위한 유틸리티가 구현되어 있다.

### 유틸리티 맵

```
utils/
├── permissions/           ─── 권한 로직 (판정, 규칙 파싱, 로딩)
├── settings/              ─── 설정 로딩/검증/MDM 통합
├── messages/              ─── 메시지 생성/정규화/변환
├── hooks/                 ─── 훅 실행 엔진
├── memory/                ─── 메모리 유틸리티
├── model/                 ─── 모델 설정/해석
├── bash/                  ─── Bash 관련 유틸리티
├── git/                   ─── Git 연동
├── github/                ─── GitHub API
├── mcp/                   ─── MCP 유틸리티
├── sandbox/               ─── 샌드박스 관리
├── shell/                 ─── 셸 유틸리티
├── skills/                ─── 스킬 유틸리티
├── plugins/               ─── 플러그인 유틸리티
├── task/                  ─── 태스크 유틸리티
├── todo/                  ─── 할일 관리
├── telemetry/             ─── 텔레메트리
├── suggestions/           ─── 제안 시스템
├── filePersistence/       ─── 파일 영속성
├── processUserInput/      ─── 사용자 입력 처리
├── background/            ─── 백그라운드 작업
├── computerUse/           ─── 컴퓨터 사용 (피처게이트)
├── deepLink/              ─── 딥링크 처리
├── dxt/                   ─── DXT 확장
├── secureStorage/         ─── 보안 저장소
├── nativeInstaller/       ─── 네이티브 설치기
├── powershell/            ─── PowerShell 유틸리티
├── claudeInChrome/        ─── Chrome 통합
├── swarm/                 ─── 스웜 모드
├── ultraplan/             ─── 울트라플랜
├── teleport/              ─── 텔레포트
│
├── fileStateCache.ts      ─── 파일 상태 캐시
├── toolResultStorage.ts   ─── 대형 도구 결과 영속화
├── attachments.ts         ─── 첨부 파일 처리
├── api.ts                 ─── API 유틸리티
├── imageValidation.ts     ─── 이미지 검증
├── imageResizer.ts        ─── 이미지 리사이즈
├── thinking.ts            ─── 사고(thinking) 설정
├── systemPromptType.ts    ─── 시스템 프롬프트 타입
├── log.ts                 ─── 로깅
├── debug.ts               ─── 디버깅
├── messageQueueManager.ts ─── 메시지 큐 관리
├── commandLifecycle.ts    ─── 명령 생명주기
├── headlessProfiler.ts    ─── 프로파일러
└── sessionStorage.ts      ─── 세션 영속성/복원/재생
```

---

## 23. 핵심 설계 패턴

### 패턴 1: Generator Streaming

```typescript
// query.ts - AsyncGenerator로 이벤트를 하나씩 yield
async function* query(params): AsyncGenerator<StreamEvent | Message, Terminal> {
  while (true) {
    // API 호출
    for await (const chunk of apiStream) {
      yield { type: 'stream', chunk }  // 청크 단위 yield
    }
    // 도구 실행 결과도 yield
    if (stopReason === 'tool_use') {
      yield toolResultMessage
      continue  // 루프 계속
    }
    return terminal  // 종료
  }
}
```

**왜?** 메모리 효율적 스트리밍. 전체 응답을 버퍼링하지 않고 청크 단위로 소비자에게 전달한다.

### 패턴 2: Feature Gates (Dead Code Elimination)

```typescript
// 빌드 시점에 피처가 꺼져 있으면 코드 자체가 번들에서 제거됨
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null

// 사용처에서 null 체크
if (reactiveCompact) {
  await reactiveCompact.compact(messages)
}
```

**왜?** Bun 번들러가 `feature()` 호출을 정적으로 평가하여 비활성 코드를 완전히 제거한다. 번들 크기를 최소화하면서도 다양한 빌드 변형을 지원한다.

### 패턴 3: Memoized Context

```typescript
// bootstrap/state.ts - 세션 수명 동안 한 번만 계산
let _sessionId: SessionId | undefined
export function getSessionId(): SessionId {
  if (!_sessionId) {
    _sessionId = crypto.randomUUID()
  }
  return _sessionId
}
```

**왜?** 세션 동안 변하지 않는 값의 반복 계산을 피한다.

### 패턴 4: Withhold & Recover

```typescript
// 복구 가능한 에러를 즉시 노출하지 않고 버퍼링
if (stopReason === 'max_output_tokens' && recoveryCount < 3) {
  // 사용자에게 보여주지 않고 자동 재시도
  state.maxOutputTokensRecoveryCount++
  continue  // 루프 반복
}
// 3회 초과 시 비로소 에러 표면화
```

**왜?** 일시적인 에러로 사용자 경험이 중단되는 것을 방지한다.

### 패턴 5: Lazy Import (순환 의존 방지)

```typescript
// tools.ts - 순환 의존을 런타임 지연 로딩으로 해결
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool

const getSendMessageTool = () =>
  require('./tools/SendMessageTool/SendMessageTool.js').SendMessageTool
```

**왜?** `tools.ts ↔ TeamCreateTool ↔ ... ↔ tools.ts` 순환을 함수 레벨 지연 로딩으로 끊는다.

### 패턴 6: Immutable State (DeepImmutable)

```typescript
type AppState = DeepImmutable<{
  settings: SettingsJson
  // ... 모든 필드가 재귀적으로 readonly
}>

// 상태 갱신은 반드시 store.setState()를 통해
store.setState({ settings: newSettings })
```

**왜?** 우발적인 상태 변이를 타입 레벨에서 차단하여 예측 가능한 상태 관리를 보장한다.

### 패턴 7: Interruption Resilience

```
매 턴 시작 전
    ↓
트랜스크립트 저장 (sessionStorage)
    ↓
쿼리 루프 실행
    ↓
사용자 인터럽트 (Ctrl+C) 발생해도
이전 턴까지의 상태는 안전하게 보존됨
```

**왜?** 사용자가 언제 중단하더라도 대화 이력이 유실되지 않는다.

### 패턴 8: Dependency Injection (테스트 모킹)

```typescript
// QueryParams에 deps 주입 가능
type QueryParams = {
  // ...
  deps?: QueryDeps  // 테스트 시 모킹용 의존성 주입
}
```

**왜?** 프로덕션 코드를 변경하지 않고도 테스트에서 API, 도구, 서비스를 교체할 수 있다.

---

## 24. 전체 데이터 흐름 요약

### 요청에서 응답까지

```
사용자 입력 ("auth.ts의 버그를 고쳐줘")
    │
    ▼
┌────────────────────────────────────────┐
│ REPL.tsx                               │
│ PromptInput → 입력 수집                │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ query.ts (AsyncGenerator)              │
│                                        │
│ [전처리]                               │
│ ├── 메모리 프리페치 시작               │
│ ├── 메시지 정규화                      │
│ └── 컨텍스트 압축 (필요 시)           │
│                                        │
│ [API 호출]                             │
│ ├── 시스템 프롬프트 조립               │
│ ├── Anthropic SDK 스트리밍 호출        │
│ └── 청크별 yield → UI 렌더링          │
│                                        │
│ [도구 실행 루프]                       │
│ ├── tool_use: "FileRead(auth.ts)"     │
│ │   └── 권한 확인 → 실행 → 결과 반환  │
│ ├── tool_use: "FileEdit(auth.ts)"     │
│ │   └── 권한 확인 → 사용자 승인 →     │
│ │       실행 → 결과 반환              │
│ └── stop_reason: "end_turn"           │
│                                        │
│ [후처리]                               │
│ ├── 세션 메모리 갱신                   │
│ ├── 비용 추적 갱신                     │
│ └── Terminal 반환                      │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ UI 렌더링                              │
│ ├── React Reconciler → DOM 트리        │
│ ├── Yoga 레이아웃 계산                 │
│ ├── 더블 버퍼링 스크린 갱신            │
│ └── ANSI 시퀀스 터미널 출력            │
└────────────────────────────────────────┘
```

### 세 가지 설계 원칙

1. **안전성(Safety)**: 10단계 권한 파이프라인, 샌드박스, 훅 기반 차단으로 위험한 작업을 다층으로 방어한다.

2. **성능(Performance)**: AsyncGenerator 스트리밍, 병렬 도구 실행, 메모이제이션, 더블 버퍼링, Dead Code Elimination으로 최적화한다.

3. **확장성(Extensibility)**: MCP 프로토콜, 플러그인, 스킬, 훅, 피처 게이트로 기능을 유연하게 확장한다.

---

## 부록: 핵심 파일 규모

| 파일 | 줄 수 | 크기 | 역할 |
|------|-------|------|------|
| `src/screens/REPL.tsx` | 5,005 | 875KB | 메인 TUI 화면 |
| `src/main.tsx` | 4,690 | 786KB | 앱 초기화/라우팅 |
| `src/query.ts` | 1,729 | 68KB | 핵심 쿼리 루프 |
| `src/QueryEngine.ts` | 1,295 | 46KB | SDK 래퍼 |
| `src/bootstrap/state.ts` | ~1,300 | 56KB | 글로벌 세션 상태 |
| `src/Tool.ts` | 792 | 29KB | 도구 인터페이스 |
| `src/tools.ts` | 389 | 17KB | 도구 등록/조합 |
| `src/commands.ts` | 754 | 25KB | 슬래시 커맨드 |
| `src/entrypoints/cli.tsx` | 302 | 13KB | CLI 진입점 |
| `src/services/mcp/client.ts` | - | 119KB | MCP 클라이언트 |
| `src/bridge/bridgeMain.ts` | - | 115KB | 브리지 코어 |
| **전체 src/** | **1,894 파일** | **~35MB** | - |
