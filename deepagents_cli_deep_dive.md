# DeepAgents CLI 심층 분석

> **deepagents-cli**의 아키텍처, 핵심 시스템, 보안 모델, 확장 메커니즘을 소스 코드 수준에서 분석한 문서입니다.
> 분석 대상: [langchain-ai/deepagents/libs/cli](https://github.com/langchain-ai/deepagents/tree/main/libs/cli) (v0.0.34, 2026-03 기준)

---

## 목차

1. [개요](#1-개요)
2. [4단계 아키텍처](#2-4단계-아키텍처)
3. [프로젝트 구조](#3-프로젝트-구조)
4. [Entry Point & CLI 파서](#4-entry-point--cli-파서)
5. [Agent 생성 & 시스템 프롬프트](#5-agent-생성--시스템-프롬프트)
6. [Interactive TUI (Textual)](#6-interactive-tui-textual)
7. [서버 아키텍처](#7-서버-아키텍처)
8. [세션 & 스레드 관리](#8-세션--스레드-관리)
9. [스킬 시스템](#9-스킬-시스템)
10. [설정 시스템](#10-설정-시스템)
11. [MCP 통합](#11-mcp-통합)
12. [도구 & 기능](#12-도구--기능)
13. [셸 명령어 실행](#13-셸-명령어-실행)
14. [보안 아키텍처](#14-보안-아키텍처)
15. [샌드박스 통합](#15-샌드박스-통합)
16. [Non-Interactive 모드](#16-non-interactive-모드)
17. [슬래시 명령어 & 커맨드 레지스트리](#17-슬래시-명령어--커맨드-레지스트리)
18. [훅 & 이벤트 시스템](#18-훅--이벤트-시스템)
19. [테마 시스템](#19-테마-시스템)
20. [개발 & 테스트](#20-개발--테스트)
21. [설정 파일 & 위치 총정리](#21-설정-파일--위치-총정리)
22. [예제 워크플로우](#22-예제-워크플로우)
23. [핵심 설계 패턴](#23-핵심-설계-패턴)
24. [정리 & 참고 자료](#24-정리--참고-자료)

---

## 1. 개요

### DeepAgents CLI란?

**deepagents-cli**는 LangChain/LangGraph 생태계 위에 구축된 **프로덕션 수준의 터미널 AI 코딩 어시스턴트**다. Claude Code에서 영감을 받아 설계되었으며, 로컬 파일 조작, 셸 명령 실행, 웹 검색, MCP 서버 연동, Human-in-the-Loop(HITL) 승인 워크플로우를 지원한다.

### 핵심 특징 요약

| 특징 | 세부 사항 |
|------|----------|
| **버전** | 0.0.34 (2026-03 기준) |
| **Python** | 3.11+ |
| **아키텍처** | 로컬 서브프로세스 모델 (localhost에 임시 LangGraph dev 서버 생성) |
| **LLM 지원** | 15개 이상 프로바이더 (Anthropic, OpenAI, Google, Groq, Ollama 등) |
| **샌드박스** | 5개 백엔드 (AgentCore, Daytona, Modal, Runloop, LangSmith) |
| **확장** | 커스텀 스킬 시스템 (SKILL.md 포맷) |
| **UI** | Interactive TUI (Textual) + Non-Interactive 헤드리스 모드 |
| **보안** | HITL 승인, Unicode 안전성 검사, MCP 신뢰 시스템 |

### 실행 모드

deepagents-cli는 **두 가지 핵심 모드**를 지원한다:

1. **Interactive 모드** (`-i`): Textual 기반 TUI로 실시간 대화
2. **Non-Interactive 모드** (`-n`): 단일 태스크를 파이프라인에서 실행

추가로 ACP(Anthropic CLI Protocol) 서버 모드도 지원하여, 외부 스크립트에서 non-interactive하게 호출할 수 있다.

---

## 2. 4단계 아키텍처

deepagents-cli의 전체 실행 흐름은 **4단계**로 구성된다:

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────┐
│  1. Startup  │───▶│ 2. Query Loop │───▶│ 3. Tool Execute │───▶│ 4. Display│
│              │    │               │    │                 │    │          │
│ - 의존성 체크  │    │ - 사용자 입력   │    │ - HITL 승인     │    │ - 응답 렌더│
│ - 설정 로드   │    │ - 컨텍스트 주입  │    │ - 파일/셸 연산   │    │ - 토큰 표시│
│ - 서버 시작   │    │ - LLM 호출     │    │ - MCP 도구 호출  │    │ - diff 표시│
│ - MCP 발견   │    │ - 미들웨어 체인  │    │ - 웹 검색/fetch  │    │ - 상태 갱신│
└─────────────┘    └──────────────┘    └─────────────────┘    └──────────┘
```

### Phase 1: Startup

CLI가 시작되면 **6개의 병렬 초기화 단계**를 수행한다:

1. **의존성 검사**: `requests`, `python-dotenv`, `tavily-python`, `textual` 등 필수 패키지 확인
2. **설정 로드**: 환경 변수, `.env` 파일, `config.toml`에서 설정값 수집
3. **MCP 서버 발견**: 3개 경로(`~/.deepagents/.mcp.json`, `.deepagents/.mcp.json`, `.mcp.json`)에서 MCP 설정 탐색
4. **모델 초기화**: 프로바이더 감지 → kwargs 조립 → `init_chat_model()` 호출
5. **세션 관리**: 기존 스레드 복원 또는 새 스레드 생성
6. **서버 기동**: LangGraph dev 서버 서브프로세스 시작 (Interactive 모드 시)

> **최적화 포인트**: `settings`와 `console` 싱글턴은 Lazy Loading으로 구현되어, `deepagents help` 같은 가벼운 명령은 ~65ms 수준의 응답 시간을 달성한다.

### Phase 2: Query Loop

사용자 입력을 받아 LLM에 전달하는 핵심 루프:

1. **입력 수집**: 멀티라인 입력, 파일 멘션 파싱, 미디어 첨부
2. **컨텍스트 주입**: `LocalContextMiddleware`가 Git 상태, 환경 정보를 시스템 프롬프트에 주입
3. **미들웨어 체인 통과**: 7개 미들웨어가 순차적으로 요청 처리
4. **LLM 호출**: SSE 스트리밍으로 응답 수신
5. **도구 호출 감지**: 응답에서 tool_call 메시지 추출

### Phase 3: Tool Execution

LLM이 도구 호출을 요청하면:

1. **HITL 검증**: 파일 쓰기, 셸 실행, 웹 접근 등은 사용자 승인 필요
2. **Unicode 안전성 검사**: URL과 파일 경로의 위험 문자 탐지
3. **도구 실행**: 승인된 도구만 실행
4. **결과 수집**: 실행 결과를 대화 컨텍스트에 추가
5. **반복**: LLM이 최종 텍스트 응답을 생성할 때까지 반복

### Phase 4: Display

최종 응답을 사용자에게 표시:

1. **마크다운 렌더링**: Rich 라이브러리로 포맷팅
2. **diff 표시**: 파일 변경 사항을 unified diff로 시각화
3. **토큰 사용량**: 입력/출력 토큰 카운트 표시
4. **세션 상태 갱신**: 체크포인트 저장, 캐시 업데이트

---

## 3. 프로젝트 구조

deepagents-cli의 전체 파일 트리와 각 파일의 역할을 정리한다. **총 47개 이상의 Python 파일**과 **4개의 서브디렉토리**로 구성된다.

```
libs/cli/
├── README.md                           # 퀵 스타트 가이드
├── CHANGELOG.md                        # 버전 히스토리 (0.0.16 → 0.0.34)
├── THREAT_MODEL.md                     # 보안 아키텍처 & 10개 활성 위협
├── DEV.md                              # 개발 가이드 (Textual devtools 설정)
├── Makefile                            # 테스트/린트/포맷 타겟
├── pyproject.toml                      # 프로젝트 설정, 의존성
├── uv.lock                             # 의존성 락 파일
│
├── deepagents_cli/                     # 메인 패키지
│   ├── __main__.py                     # python -m 진입점
│   ├── main.py                         # CLI 인자 파서 & 커맨드 디스패처
│   ├── app.py                          # Textual TUI 애플리케이션
│   ├── agent.py                        # Agent 생성, 시스템 프롬프트
│   ├── server_graph.py                 # LangGraph 서버 초기화
│   ├── server.py                       # LangGraph dev 서버 생명주기
│   ├── server_manager.py               # 서버 시작 오케스트레이션
│   ├── remote_client.py                # HTTP+SSE 클라이언트
│   ├── ui.py                           # 도움말 화면 & argparse 유틸
│   │
│   ├── config.py                       # 설정, 환경 변수, 모델 생성
│   ├── model_config.py                 # TOML 기반 모델/프로바이더 설정
│   ├── _server_config.py               # CLI↔서버 IPC (환경 변수)
│   ├── _env_vars.py                    # 환경 변수 상수 레지스트리
│   ├── _cli_context.py                 # CLIContext TypedDict
│   ├── configurable_model.py           # 런타임 모델 선택 미들웨어
│   ├── token_state.py                  # 토큰 사용량 추적 미들웨어
│   ├── ask_user.py                     # ask_user 도구 미들웨어
│   ├── local_context.py                # Git/환경 컨텍스트 주입 미들웨어
│   │
│   ├── sessions.py                     # 스레드 관리, 체크포인트 쿼리
│   ├── non_interactive.py              # 헤드리스 모드 실행
│   ├── input.py                        # 파일 멘션 파싱, 미디어 추적
│   ├── output.py                       # JSON 출력 포맷팅
│   ├── file_ops.py                     # 파일 연산 추적, diff 생성
│   ├── tools.py                        # web_search & fetch_url
│   ├── editor.py                       # 외부 에디터 ($VISUAL/$EDITOR)
│   ├── clipboard.py                    # 클립보드 연산
│   ├── hooks.py                        # 이벤트 훅 디스패치
│   ├── project_utils.py                # Git 루트 탐지, AGENTS.md 위치
│   ├── formatting.py                   # format_duration() 헬퍼
│   ├── unicode_security.py             # BiDi/유니코드 호모글리프 탐지
│   ├── mcp_trust.py                    # SHA-256 핑거프린트 신뢰
│   ├── theme.py                        # 테마 색상, 다크/라이트 팔레트
│   ├── subagents.py                    # AGENTS.md 로딩, 위임
│   ├── offload.py                      # 샌드박스로 대화 오프로드
│   ├── update_check.py                 # CLI 업데이트 확인
│   ├── mcp_tools.py                    # MCP 서버 발견, 검증, 로딩
│   ├── command_registry.py             # 슬래시 명령어 정의
│   ├── media_utils.py                  # 이미지/비디오 유틸리티
│   ├── textual_adapter.py              # LangGraph ↔ Textual 브릿지
│   │
│   ├── system_prompt.md                # Claude 시스템 지침
│   ├── default_agent_prompt.md         # 기본 에이전트 컨텍스트 템플릿
│   ├── app.tcss                        # Textual CSS 스타일링
│   │
│   ├── skills/                         # 스킬 발견 & 호출
│   │   ├── __init__.py                 # Public API
│   │   ├── commands.py                 # list/create/info/delete 서브커맨드
│   │   ├── load.py                     # 7개 디렉토리 스킬 탐색
│   │   └── invocation.py               # 스킬 래핑, 메타데이터 삽입
│   │
│   ├── integrations/                   # 샌드박스 & 프로바이더 팩토리
│   │   ├── sandbox_factory.py          # 5개 샌드박스 프로바이더 컨텍스트 매니저
│   │   └── sandbox_provider.py         # SandboxProvider 추상 베이스 클래스
│   │
│   ├── built_in_skills/                # 기본 내장 스킬
│   │   ├── remember/SKILL.md           # 영구 메모리 스킬
│   │   └── skill-creator/SKILL.md      # 스킬 생성 가이드 (19KB)
│   │
│   └── widgets/                        # Textual UI 컴포넌트 (18개 파일)
│       ├── chat_input.py               # 멀티라인 입력, 자동완성, 히스토리 (68KB)
│       ├── messages.py                 # 메시지 렌더링, 가상화 (60KB)
│       ├── thread_selector.py          # 세션/스레드 전환 (65KB)
│       ├── model_selector.py           # 모델 피커, 퍼지 검색
│       ├── approval.py                 # 인라인 승인 다이얼로그
│       ├── ask_user.py                 # 사용자 질문 프롬프트
│       ├── autocomplete.py             # 명령/스킬 자동완성
│       ├── diff.py                     # Unified diff 표시
│       ├── mcp_viewer.py               # MCP 도구 브라우저
│       ├── loading.py                  # 로딩 스피너
│       ├── message_store.py            # 인메모리 메시지 저장 & 캐싱
│       └── ...                         # (tool_widgets, status, theme_selector 등)
│
├── examples/skills/                    # 예제 스킬
│   ├── arxiv-search/                   # arXiv 논문 검색
│   ├── langgraph-docs/                 # LangGraph 문서 참조
│   ├── skill-creator/                  # 스킬 생성 템플릿
│   └── web-research/                   # 웹 리서치 워크플로우
│
├── tests/                              # 테스트 모음
│   ├── unit_tests/                     # 유닛 테스트
│   ├── integration_tests/              # 통합 테스트 (30초 타임아웃)
│   └── README.md
│
└── scripts/                            # 유틸리티 스크립트
```

### 파일 크기 분석

가장 큰 소스 파일들은 UI 위젯 계층에 집중되어 있다:

| 파일 | 크기 | 역할 |
|------|------|------|
| `widgets/chat_input.py` | ~68KB | 입력 위젯 (멀티라인, 히스토리, 자동완성) |
| `widgets/thread_selector.py` | ~65KB | 세션 브라우저 |
| `widgets/messages.py` | ~60KB | 메시지 렌더링 & 가상화 |
| `skills/commands.py` | ~37KB | 스킬 CLI 명령어 |
| `integrations/sandbox_factory.py` | ~29KB | 5개 샌드박스 프로바이더 팩토리 |
| `built_in_skills/skill-creator/SKILL.md` | ~19KB | 스킬 생성 가이드 |

---

## 4. Entry Point & CLI 파서

### 진입점

deepagents-cli는 **3가지 방법**으로 실행할 수 있다:

```bash
deepagents           # 메인 CLI (pyproject.toml의 scripts entry)
deepagents-cli       # 별칭
python -m deepagents_cli  # 모듈 실행 (__main__.py)
```

### main.py: CLI 인자 파서 & 커맨드 디스패처

`main.py`는 전체 CLI의 오케스트레이터로, argparse 기반의 인자 파서와 5개의 서브커맨드를 정의한다.

#### 서브커맨드

| 서브커맨드 | 설명 |
|-----------|------|
| `help` | 도움말 표시 |
| `agents` | 서브에이전트 관리 |
| `skills` | 스킬 리스트/생성/조회/삭제 |
| `threads` | 세션 브라우저 |
| `update` | CLI 업데이트 확인 |

#### 핵심 플래그

```bash
deepagents -i                    # Interactive TUI 모드
deepagents -n                    # Non-Interactive 헤드리스 모드
deepagents -r                    # 가장 최근 스레드 재개
deepagents -r <thread-id>        # 특정 스레드 재개
deepagents -t <thread-id>        # 스레드 ID 직접 지정
deepagents -m <model>            # 모델 오버라이드
deepagents -p '{"temperature": 0.7}'  # 모델 파라미터 (JSON)
```

#### 시작 흐름

```python
def main():
    # 1. argparse 설정 & 파싱
    parser = create_parser()
    args = parser.parse_args()

    # 2. 의존성 검사 (requests, dotenv, tavily 등)
    check_dependencies()

    # 3. 설정 로드 (Lazy Singleton)
    settings = get_settings()

    # 4. stdin 처리 (파이프 입력 시 TTY 복원)
    stdin_content = read_stdin_if_piped()

    # 5. 모드 분기
    if args.non_interactive:
        run_non_interactive(task, agent)
    else:
        app = DeepAgentsApp(...)
        app.run()

    # 6. 종료 시 통계 표시
    display_session_stats()
```

> **주목할 점**: stdin이 파이프로 전달되면 (`echo "fix this bug" | deepagents -n`), CLI는 파이프 내용을 읽은 뒤 TTY를 복원하여 interactive 프롬프트를 정상 작동시킨다.

---

## 5. Agent 생성 & 시스템 프롬프트

### create_cli_agent() 함수

`agent.py`의 `create_cli_agent()`는 LangGraph 에이전트를 조립하는 핵심 함수다. 이 함수가 반환하는 에이전트는 **7개의 미들웨어**로 감싸져 있다.

### 미들웨어 스택

미들웨어는 **요청 전처리 → LLM 호출 → 응답 후처리**의 체인으로 동작한다. 스택 순서는 다음과 같다:

```
요청 ──▶ [1] ConfigurableModelMiddleware  ← 런타임 모델 전환
     ──▶ [2] TokenStateMiddleware          ← 토큰 사용량 추적
     ──▶ [3] AskUserMiddleware             ← 사용자 질문 처리
     ──▶ [4] MemoryMiddleware              ← 영구 마크다운 메모리
     ──▶ [5] SkillsMiddleware              ← 커스텀 스킬 호출
     ──▶ [6] LocalContextMiddleware        ← Git/환경 컨텍스트 주입
     ──▶ [7] ShellAllowListMiddleware      ← 셸 명령어 검증 (non-blocking)
     ──▶ LLM 호출
```

#### 각 미들웨어의 역할

| # | 미들웨어 | 파일 | 역할 |
|---|---------|------|------|
| 1 | `ConfigurableModelMiddleware` | `configurable_model.py` | `/model` 명령으로 런타임 중 LLM 교체 |
| 2 | `TokenStateMiddleware` | `token_state.py` | 입력/출력 토큰 카운트 누적 |
| 3 | `AskUserMiddleware` | `ask_user.py` | `ask_user` 도구 호출 시 사용자에게 질문 |
| 4 | `MemoryMiddleware` | (deepagents 코어) | 영구 마크다운 파일 기반 메모리 |
| 5 | `SkillsMiddleware` | (deepagents 코어) | `/skill:<name>` 명령으로 스킬 호출 |
| 6 | `LocalContextMiddleware` | `local_context.py` | 현재 Git 브랜치, 환경 정보를 시스템 프롬프트에 주입 |
| 7 | `ShellAllowListMiddleware` | (deepagents 코어) | 허용되지 않은 셸 명령을 에러 메시지로 차단 (non-blocking) |

### 시스템 프롬프트 구성

시스템 프롬프트는 다음 요소들을 조합하여 동적으로 생성된다:

1. **모델 아이덴티티**: 사용 중인 모델명과 컨텍스트 윈도우 크기
2. **작업 디렉토리**: 로컬 경로 또는 샌드박스 경로
3. **모드 가이드**: Interactive vs Headless 모드별 지침
4. **파일 경로 처리**: 절대/상대 경로 규칙
5. **안전한 도구 사용**: 파일 쓰기, 셸 실행 시 주의사항
6. **Makefile 컨텍스트**: 프로젝트 루트의 Makefile 첫 20줄이 프롬프트에 주입됨

### HITL(Human-in-the-Loop) 인터럽트

다음 도구 호출은 **자동 승인되지 않고** 사용자 확인을 요구한다:

- **파일 연산**: 파일 쓰기, 편집, 삭제
- **웹 접근**: URL fetch, 웹 검색
- **셸 실행**: 모든 셸 명령어
- **태스크 위임**: 서브에이전트에 작업 위임

> **auto-approve 모드**: 사용자가 명시적으로 opt-in하면 모든 도구 호출을 자동 승인할 수 있다. 이는 보안 위험을 수반하므로 명시적 동의가 필요하다.

### 서브에이전트 지원

```
.deepagents/agents/
├── code-reviewer/
│   └── AGENTS.md          # 코드 리뷰 전용 에이전트
├── test-writer/
│   └── AGENTS.md          # 테스트 작성 전용 에이전트
└── ...
```

- `.deepagents/agents/*/AGENTS.md` 파일에서 커스텀 에이전트를 로드
- `AGENTS.md`의 본문이 서브에이전트의 `system_prompt`로 직접 사용됨
- 원격 비동기 서브에이전트도 LangGraph 배포를 통해 지원
- 부모 에이전트의 shell allow-list가 서브에이전트에도 적용됨

---

## 6. Interactive TUI (Textual)

### DeepAgentsApp 클래스

`app.py`의 `DeepAgentsApp`은 [Textual](https://textual.textualize.io/) 프레임워크 기반의 메인 TUI 애플리케이션이다.

### UI 컴포넌트 구성

```
┌──────────────────────────────────────────────────┐
│                 DeepAgentsApp                     │
├──────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────┐  │
│  │          Messages (가상화 스크롤)            │  │
│  │  - 사용자 메시지                             │  │
│  │  - AI 응답 (마크다운 렌더링)                  │  │
│  │  - 도구 호출 결과 (diff, 코드 등)             │  │
│  │  - 승인 요청 다이얼로그                       │  │
│  └────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────┐  │
│  │  ChatInput (멀티라인, 자동완성, 히스토리)      │  │
│  └────────────────────────────────────────────┘  │
│  [Status: 토큰 사용량 | 모델명 | 스레드 ID]       │
└──────────────────────────────────────────────────┘
```

### 18개 위젯 파일

| 위젯 | 파일 | 핵심 기능 |
|------|------|----------|
| **ChatInput** | `chat_input.py` (68KB) | 멀티라인 편집, 명령어 자동완성, 입력 히스토리, 파일 첨부 |
| **Messages** | `messages.py` (60KB) | 메시지 렌더링, 가상화(hydration)로 효율적 스크롤백 |
| **ThreadSelector** | `thread_selector.py` (65KB) | 세션 브라우저, 스레드 전환, 검색 |
| **ModelSelector** | `model_selector.py` | 모델 피커, 퍼지 검색 |
| **Approval** | `approval.py` | HITL 승인 다이얼로그 |
| **AskUser** | `ask_user.py` | 사용자 질문 프롬프트 |
| **Autocomplete** | `autocomplete.py` | 명령어/스킬 자동완성 |
| **Diff** | `diff.py` | Unified diff 시각화 |
| **McpViewer** | `mcp_viewer.py` | MCP 도구 브라우저 |
| **Loading** | `loading.py` | 로딩 스피너 |
| **MessageStore** | `message_store.py` | 인메모리 메시지 캐싱 |

### 주요 동작 특성

#### 메시지 가상화(Virtualization)

대규모 대화에서도 UI 반응성을 유지하기 위해, 화면에 보이지 않는 메시지는 **경량 플레이스홀더**로 대체되고, 스크롤 시 실제 콘텐츠로 **hydration** 된다.

#### 타이핑 인식 승인 지연

도구 호출 승인 다이얼로그는 사용자가 타이핑 중일 때 즉시 표시되지 않는다. **2초 idle 임계값**을 적용하여, 사용자가 입력을 멈춘 후에만 승인 다이얼로그를 표시한다. 이는 타이핑 도중 갑자기 나타나는 모달이 UX를 방해하는 것을 방지한다.

#### Deferred Action Queue

에이전트 실행 중이나 서버 연결 대기 중에 발생하는 사용자 액션은 **큐에 적재**되어, 시스템이 준비되면 순차적으로 처리된다. 이로써 race condition을 방지한다.

#### 성능 최적화

- **지연 임포트**: `markdown`, `pygments` 등 무거운 모듈은 첫 번째 렌더링 시점에 임포트하여, 초기 화면 표시(first paint)를 차단하지 않는다.
- **iTerm2 워크어라운드**: OSC 1337 이스케이프 시퀀스로 cursor-guide 관련 호환성 문제를 해결한다.

---

## 7. 서버 아키텍처

### 클라이언트-서버 패턴

deepagents-cli는 **서브프로세스 기반의 클라이언트-서버 모델**을 사용한다. CLI(클라이언트)가 임시 LangGraph dev 서버를 로컬에 생성하고, HTTP+SSE로 통신한다.

```
┌─────────────┐     HTTP+SSE      ┌──────────────────┐
│ CLI Client   │ ◀──────────────▶ │ LangGraph Server  │
│ (Textual UI) │   127.0.0.1:port │ (Subprocess)      │
│              │                   │                   │
│ RemoteAgent  │                   │ server_graph.py   │
│ (SSE Client) │                   │ SQLite Checkpoint │
└─────────────┘                   └──────────────────┘
```

### 서버 생명주기

#### 1. 워크스페이스 스캐폴딩 (`server_manager.py`)

서버 시작 전, 임시 디렉토리에 최소한의 프로젝트를 생성한다:

- `server_graph.py`: LangGraph 진입점 (import-time 초기화)
- SQLite 체크포인터 모듈
- 최소 `pyproject.toml`

#### 2. 서버 시작 (`server.py`)

```python
# 서브프로세스로 langgraph dev 서버 시작
subprocess.Popen(
    ["langgraph", "dev", "--port", str(port), "--host", "127.0.0.1"],
    env={**os.environ, **server_config_env_vars}
)
```

#### 3. 헬스 모니터링

- **적응형 폴링**: 로컬 0.1초 / 원격 0.3초 간격
- **60초 타임아웃**: 시작 실패 시 조기 종료 감지
- **포트 자동 발견**: 바인딩 충돌 시 다른 포트로 시도

#### 4. 정상 종료

CLI 종료 시 서버 프로세스를 gracefully shutdown하며, 필요 시 재시작도 지원한다.

### ServerConfig 데이터클래스

CLI와 서버 간의 설정 전달은 `DA_SERVER_*` 환경 변수를 통해 이루어진다:

```python
@dataclass
class ServerConfig:
    model_name: str           # 사용 모델
    model_params: dict        # 모델 파라미터 (JSON)
    auto_approve: bool        # 자동 승인 모드
    interrupt_settings: dict  # HITL 인터럽트 설정
    shell_access: str         # 셸 접근 제어
    sandbox_type: str         # 샌드박스 유형
    sandbox_id: str           # 샌드박스 ID
    sandbox_setup: str        # 샌드박스 설정 스크립트
    mcp_settings: dict        # MCP 설정
    working_dir: str          # 작업 디렉토리
    project_root: str         # 프로젝트 루트
    # ... 20개 이상의 필드
```

**직렬화 방식**:
- `dict`/`list` → JSON 문자열
- 문자열 리스트 → 콤마 구분
- `str`/`bool` → 직접 전달

### 로컬 Pregel 에이전트 (폴백)

LangGraph 서버 시작에 실패하면, **로컬 Pregel 에이전트**로 폴백한다:

| 특성 | 로컬 Pregel | 원격 LangGraph 서버 |
|------|------------|-------------------|
| 시작 속도 | 빠름 | 서버 부팅 대기 필요 |
| 서브에이전트 | 동기식만 | 비동기 지원 |
| 프로세스 | 단일 | 분리 (서브프로세스) |
| 장애 격리 | 없음 | 서버 크래시가 CLI에 영향 없음 |

---

## 8. 세션 & 스레드 관리

### 개요

모든 대화는 **스레드(thread)**로 관리된다. 각 스레드는 고유 ID를 가지며, SQLite 체크포인트에 상태가 저장된다.

### sessions.py: 핵심 연산

| 연산 | 설명 |
|------|------|
| **스레드 목록 조회** | 에이전트명 & Git 브랜치로 필터링 |
| **메시지 수 추출** | 체크포인트 blob에서 메시지 카운트 파싱 |
| **초기 프롬프트 복구** | 대화 히스토리에서 첫 번째 사용자 메시지 추출 |
| **스레드 삭제** | 특정 스레드 제거 |
| **스레드 조회** | ID로 스레드 상태 조회 |

### 캐싱 전략

성능을 위해 **3단계 캐시**를 운영한다:

```python
# 1. 메시지 수 캐시 (LRU, 4096 엔트리)
@lru_cache(maxsize=4096)
def get_message_count(thread_id, checkpoint_id):
    ...

# 2. 초기 프롬프트 캐시 (LRU, 4096 엔트리)
@lru_cache(maxsize=4096)
def get_initial_prompt(thread_id, checkpoint_id):
    ...

# 3. 최근 스레드 캐시 (LRU, 16 조합)
@lru_cache(maxsize=16)
def get_recent_threads(agent_name, branch, limit):
    ...
```

**신선도(Freshness) 보장**: 체크포인트 ID를 캐시 키에 포함시켜, 스레드 상태가 변경되면 자동으로 캐시가 무효화된다.

### 스레드 목록 출력

두 가지 형식을 지원한다:

1. **텍스트 형식**: Rich 라이브러리로 렌더링된 테이블
2. **JSON 형식**: 프로그래밍 통합용 구조화 데이터

```bash
deepagents threads             # Rich 테이블
deepagents threads --json      # JSON 출력
```

---

## 9. 스킬 시스템

### 스킬이란?

스킬은 **재사용 가능한 지식 모듈**로, SKILL.md 파일에 정의된 지침과 리소스의 묶음이다. 에이전트가 특정 작업을 수행할 때 관련 스킬이 시스템 프롬프트에 주입된다.

> **도구(Tool) vs 스킬(Skill)**: 도구는 실행 가능한 코드(함수)이고, 스킬은 에이전트의 행동을 안내하는 지침(텍스트)이다. 스킬은 도구를 어떻게 사용할지 가르치는 역할을 한다.

### 스킬 발견 경로 (7개 소스, 우선순위 순)

```
우선순위 높음 ──────────────────────────────────── 낮음

1. 빌트인         <package>/built_in_skills/
2. 사용자 에이전트  ~/.agents/<agent-name>/
3. 사용자 전역     ~/.agents/skills/
4. 사용자 메인     ~/.agents/agent/skills/
5. 프로젝트 에이전트 .agents/<agent-name>/
6. 프로젝트 메인    .agents/agent/skills/
7. 프로젝트        .deepagents/skills/
```

높은 우선순위의 스킬이 같은 이름의 낮은 우선순위 스킬을 **오버라이드**한다.

**보안**: 심볼릭 링크 순회를 방지하기 위해 resolved path 검증을 수행한다.

### SKILL.md 포맷 명세

```yaml
---
name: skill-name                        # 최대 64자, 소문자 영숫자 + 하이픈
description: "무엇을 하는지 AND 언제 트리거되는지"  # 트리거 조건이 매우 중요!
license: MIT                             # 선택
compatibility: deepagents-cli            # 선택
allowed-tools: ["tool1", "tool2"]        # 선택, 스킬이 사용할 수 있는 도구 제한
---

# 스킬 제목

## Overview
스킬의 목적과 컨텍스트를 설명한다.

## Best Practices
- 실천 사항 1: 설명
- 실천 사항 2: 설명

## Process
1. 첫 번째 단계 (명령형으로 작성)
2. 두 번째 단계
3. ...

## Common Pitfalls
- 함정 1: 왜 피해야 하는지
- 함정 2: 왜 피해야 하는지
```

> **핵심**: `description` 필드가 스킬의 **자동 트리거 조건**을 결정한다. "사용자가 X를 요청할 때"와 같은 트리거 구문을 포함해야 한다.

### 선택적 리소스 디렉토리

```
my-skill/
├── SKILL.md              # 필수: 스킬 정의
├── scripts/              # 선택: 실행 가능한 코드 (반복적/취약한 작업용)
├── references/           # 선택: 추가 문서 (필요 시 로드)
└── assets/               # 선택: 출력 파일 (템플릿, 아이콘 등)
```

### 스킬 CLI 명령어

```bash
deepagents skills list                # 사용 가능한 스킬 목록 (소스 표시)
deepagents skills create <name>       # 새 SKILL.md 생성
deepagents skills info <name>         # 메타데이터 & 내용 표시
deepagents skills delete <name>       # 삭제 (안전성 검증 포함)

# 옵션
--force    # 확인 건너뛰기
--json     # JSON 출력
```

### 빌트인 스킬

#### 1. `remember` - 영구 메모리

세션 간 지속되는 마크다운 기반 메모리 시스템. 사용자가 `/remember` 명령으로 메모를 저장하면, 이후 세션에서도 해당 메모를 참조할 수 있다.

#### 2. `skill-creator` - 스킬 생성 가이드

19KB 분량의 포괄적 스킬 설계 가이드:
- Progressive disclosure 디자인 (메타데이터 → 본문 → 리소스)
- 재사용 가능한 리소스 작성 모범 사례
- 검증 체크리스트
- 적절한 자유도(degrees of freedom) 설정 가이드

### 스킬 호출 메커니즘 (`invocation.py`)

스킬이 트리거되면:

1. SKILL.md 내용을 **envelope**로 래핑
2. 메타데이터(이름, 소스, allowed-tools)를 삽입
3. 시스템 프롬프트에 주입
4. 에이전트가 스킬 지침에 따라 행동

---

## 10. 설정 시스템

### 설정 소스 우선순위

설정값은 다음 순서로 해석된다 (높은 우선순위가 낮은 것을 오버라이드):

```
1. 환경 변수 (shell-exported)              ← 최고 우선순위
2. DEEPAGENTS_CLI_* 접두사 환경 변수 (.env)
3. 접두사 없는 환경 변수 (.env)
4. 글로벌 ~/.deepagents/.env              ← 최저 우선순위
```

### config.py: Settings 클래스

```python
class Settings:
    """Lazy 싱글턴 - 첫 접근 시까지 expensive 연산을 지연"""

    # API 키 감지
    anthropic_api_key: str | None
    openai_api_key: str | None
    google_api_key: str | None
    tavily_api_key: str | None
    nvidia_api_key: str | None

    # LangSmith 설정
    langsmith_api_key: str | None
    langsmith_project: str | None

    # 프로젝트 감지
    project_root: str           # .git 기반 탐지
    shell_allow_list: list[str]
    extra_skill_dirs: list[str]
    model_preference: str       # 기본 모델
```

### 모델 생성 파이프라인

모델 인스턴스 생성은 **3단계 파이프라인**으로 진행된다:

```
1. 프로바이더 감지     "claude-opus-4.1" → Anthropic
                      "gpt-4o"         → OpenAI
                      "gemini-2.0"     → Google

2. kwargs 조립        config.toml + 환경 변수 + CLI 인자에서 수집
                      temperature, max_tokens, api_key 등

3. 모델 인스턴스화     init_chat_model() 또는 커스텀 서브클래스
```

### model_config.py: TOML 기반 설정

`~/.deepagents/config.toml`에서 프로바이더, 모델, 테마를 설정한다:

```toml
# 프로바이더 설정
[providers.anthropic]
enabled = true
models = ["claude-opus-4.1-20250805"]
api_key_env = "ANTHROPIC_API_KEY"

[providers.openai]
enabled = true
models = ["gpt-4o"]
class_path = "openai.ChatOpenAI"        # 커스텀 인스턴스화 (선택)

# 모델별 오버라이드
[models."gpt-4o".overrides]
temperature = 0.7
max_tokens = 8000

# 스레드 테이블 설정
[threads]
columns = ["timestamp", "status", "message"]
timestamp_format = "local"
sort_key = "created_at"

# 커스텀 테마
[themes.my-custom-theme]
label = "My Custom"
dark = true
primary = "#ff6b6b"
```

### 모델 발견 메커니즘

설치된 LangChain 프로바이더에서 **동적으로 모델을 발견**한다:

1. 설치된 `langchain-*` 패키지에서 프로바이더 목록 수집
2. tool calling + text I/O를 지원하는 모델만 필터링
3. config.toml의 오버라이드 적용
4. 퍼지 검색으로 모델 선택 UI 제공

### 프로바이더 자격증명 검증

3단계 자격증명 확인:

```
1단계: config.toml의 api_key_env 변수 확인
2단계: 하드코딩된 프로바이더 매핑 (ANTHROPIC_API_KEY 등)
3단계: 지연된 확인 (모델 첫 호출 시)
```

> **빈 문자열 처리**: 빈 문자열(`""`)은 명시적으로 "키 없음"을 의미하며, 폴백 값 사용을 억제한다.

---

## 11. MCP 통합

### MCP(Model Context Protocol)란?

MCP는 AI 모델이 외부 도구와 데이터 소스에 접근할 수 있게 해주는 프로토콜이다. deepagents-cli는 MCP 서버를 자동으로 발견하고, 에이전트가 사용할 수 있는 도구로 노출한다.

### 지원 전송 방식

| 전송 방식 | 설명 | 예시 |
|----------|------|------|
| **Stdio** | 로컬 프로세스 기반 | `node server.js` |
| **SSE** | Server-Sent Events (HTTP 스트리밍) | `http://localhost:3000/sse` |
| **HTTP** | 표준 HTTP 요청 | `http://localhost:3000` |

### 발견 순서 (우선순위)

```
1. ~/.deepagents/.mcp.json       ← 사용자 레벨 (최고 우선순위)
2. .deepagents/.mcp.json         ← 프로젝트 서브디렉토리
3. .mcp.json                     ← 프로젝트 루트 (최저 우선순위)
```

### 설정 형식

```json
{
  "servers": {
    "my-stdio-server": {
      "command": "node",
      "args": ["server.js"],
      "type": "stdio"
    },
    "my-http-server": {
      "url": "http://localhost:3000",
      "type": "http"
    },
    "my-sse-server": {
      "url": "http://localhost:3000/sse",
      "type": "sse"
    }
  }
}
```

### 신뢰 시스템 (`mcp_trust.py`)

프로젝트 레벨의 stdio MCP 서버는 **로컬 코드를 실행**하므로 보안 위험이 있다. 이를 관리하기 위해 **SHA-256 핑거프린트 기반 신뢰 시스템**을 사용한다.

#### 신뢰 흐름

```
1. 프로젝트 .mcp.json 발견
2. 파일 내용의 SHA-256 해시 계산
3. ~/.deepagents/config.toml의 [mcp_trust.projects] 확인
4. 핑거프린트 일치 → 자동 허용
   핑거프린트 불일치 → 사용자에게 명시적 승인 요청
5. 승인 시 핑거프린트 저장
```

#### 핑거프린트 관리

```python
# 신뢰 부여
grant_project_mcp_trust(project_path, config_content)

# 신뢰 철회
revoke_project_mcp_trust(project_path)

# 설정 변경 시 → 핑거프린트 불일치 → 재승인 필요
```

### 안전성 기능

- **검증 & 헬스 체크**: 세션 설정 전 서버 상태 확인
- **타임아웃**: 원격 서버에 2초 HEAD 요청
- **에러 격리**: 한 MCP 서버의 오류가 다른 서버에 전파되지 않음
- **명시적 vs 자동발견**: 명시적 설정의 오류는 fatal, 자동발견된 설정의 오류는 warning

---

## 12. 도구 & 기능

### 빌트인 도구 (`tools.py`)

#### web_search()

```python
def web_search(
    query: str,
    max_results: int = 5,
    topic: str = "general",          # "general" | "news"
    include_raw_content: bool = False
) -> dict:
    """Tavily API를 통한 웹 검색"""
    # 반환값: titles, URLs, content excerpts, relevance scores
```

- Tavily API 키 필요 (`TAVILY_API_KEY`)
- 의존성 누락 시 graceful 에러 메시지

#### fetch_url()

```python
def fetch_url(
    url: str,
    timeout: int = 30
) -> dict:
    """HTML을 가져와 마크다운으로 변환"""
    # 반환값: success, final_url, markdown_content, http_status, content_length
```

- HTML → Markdown 변환
- 리다이렉트 추적 (final_url 반환)
- 의존성 누락 시 graceful 처리

### 파일 연산 추적 (`file_ops.py`)

#### FileOpTracker 클래스

파일 연산의 전체 생명주기를 추적한다:

```
시작(start) ──▶ 완료(complete) ──▶ HITL 승인(approve)
    │                │                    │
    ▼                ▼                    ▼
  경로 기록      수정 전/후 캡처      승인 후 실행
```

#### 승인 미리보기

HITL 다이얼로그에 표시되는 정보:

| 항목 | 설명 |
|------|------|
| 파일 경로 | 대상 파일의 절대/상대 경로 |
| 액션 설명 | write, edit, delete 등 |
| 라인 변경 수 | 추가/삭제된 라인 수 |
| Unified diff | 설정 가능한 컨텍스트 라인 수의 diff |

---

## 13. 셸 명령어 실행

### Allow-list 전략

deepagents-cli는 **화이트리스트 기반**의 셸 명령어 실행 제어를 사용한다.

#### ShellAllowListMiddleware

```
사용자 입력 ──▶ 패턴 차단 ──▶ 명령어 파싱 ──▶ Allow-list 확인 ──▶ 실행/거부
                  │
                  ▼
             위험 패턴 즉시 차단:
             $() , backticks, |, >, &, ${}
```

#### 동작 방식

1. **패턴 차단**: `$()`, 백틱, `|`, `>`, `&`, `${}` 같은 위험 패턴을 **파싱 전에** 차단
2. **명령어 파싱**: 허용된 명령어 목록과 대조
3. **거부 처리**: 비허용 명령을 **에러 메시지로 반환** (non-blocking, 에이전트 실행을 중단하지 않음)

#### 안전한 기본 명령어 세트

```
읽기 전용: ls, cat, grep, find, head, tail, less, pwd,
          git status, git log, git diff, git branch,
          wc, file, which, env, echo, date, tree
```

#### 셸 접근 레벨

| 레벨 | 설명 |
|------|------|
| 미설정 | 셸 비활성화, 다른 도구는 자동 승인 |
| `"recommended"` | 안전 명령어만 허용, 다른 도구 자동 승인 |
| `"all"` | 모든 명령어 허용 (`SHELL_ALLOW_ALL` sentinel) |

```bash
deepagents --shell-allow-list recommended   # 권장 안전 명령어
deepagents --shell-allow-list all           # 모든 명령어 (주의!)
```

---

## 14. 보안 아키텍처

### HITL을 핵심 보안 경계로

deepagents-cli의 보안 모델은 **Human-in-the-Loop(HITL)**을 핵심 경계로 설정한다. 모든 부작용(side-effect)을 가진 도구 호출은 기본적으로 사용자 승인을 요구한다.

### 위협 모델 (THREAT_MODEL.md)

10개의 활성 위협이 심각도별로 분류되어 있다:

#### Medium 심각도

| ID | 위협 | 통제 수단 |
|----|------|----------|
| **T1** | **프롬프트 인젝션**: 웹에서 가져온 콘텐츠에 적대적 지시 포함 | Interactive 모드에서 HITL 승인 |
| **T2** | **셸 Allow-list 우회**: `--shell-allow-list all`이 패턴 검사 비활성화 | T1과 결합 시 Non-Interactive에서 RCE 가능 |
| **T6** | **인증 없는 Dev 서버**: localhost 포트 열거 가능, 인증 없음 | 로컬 프로세스가 상태 주입 가능 |

#### Low 심각도

| ID | 위협 | 통제 수단 |
|----|------|----------|
| **T3** | **유니코드 호모글리프**: 시각적으로 유사한 문자로 URL 위장 | 탐지 + 경고 시스템 |
| **T4** | **Auto-Approve 모드**: 모든 도구를 자동 승인 | 명시적 사용자 opt-in 필요 |
| **T5** | **SQLite 변조**: 체크포인트 DB 조작 | 파일시스템 쓰기 권한 필요 (계정 침해 전제) |
| **T7** | **Makefile 인젝션**: Makefile 첫 20줄이 프롬프트에 주입 | 프로젝트 파일 쓰기 권한 필요 |
| **T8** | **서브에이전트 본문 인젝션**: AGENTS.md가 system_prompt로 직접 사용 | 프로젝트 파일 쓰기 권한 필요 |
| **T9** | **class_path 코드 실행**: importlib로 임의 모듈 로드 | 의도적 설계 (사용자 제어 설정) |
| **T10** | **MCP ENV 필터링 없음**: ENV dict 키가 검증되지 않음 | PATH, LD_PRELOAD 주입 가능 |

### Unicode 보안 (`unicode_security.py`)

#### 위험 유니코드 탐지

```python
# 탐지 대상
DANGEROUS_UNICODE = {
    "BiDi 제어": "U+202A-202F, U+2066-206A",
    "Zero-width": "ZWJ, ZWNJ, ZWSP",
    "불가시 포맷팅": "Soft Hyphen, Word Joiner 등"
}
```

#### URL 안전성 분석

```python
def check_url_safety(url: str) -> list[str]:
    """URL의 보안 위험을 분석하여 경고 목록을 반환"""
    warnings = []

    # 1. URL 내 숨겨진 유니코드 탐지
    if has_hidden_unicode(url):
        warnings.append("Hidden Unicode characters detected")

    # 2. Punycode 디코딩 (IDN 도메인)
    if is_punycode(domain):
        warnings.append(f"IDN domain: {decoded_domain}")

    # 3. 스크립트 혼합 (여러 문자 체계)
    if has_mixed_scripts(domain):
        warnings.append("Mixed scripts in domain")

    # 4. 혼동 가능 문자 (키릴 'а' vs 라틴 'a')
    if has_confusables(domain):
        warnings.append("Confusable characters detected")

    return warnings
```

#### 사용자 피드백 함수

| 함수 | 역할 |
|------|------|
| `check_url_safety()` | 경고 목록 반환 |
| `strip_dangerous_unicode()` | 위험 문자 제거 |
| `render_with_unicode_markers()` | 숨겨진 문자를 레이블로 표시 |

### 설계 철학

| 원칙 | 설명 |
|------|------|
| **HITL이 주 보안 경계** | 모든 side-effect 도구는 승인 필요 |
| **서브프로세스 중심** | 임시 로컬 서버, 인증 없음 (로컬 신뢰) |
| **설정은 사용자 코드** | class_path & MCP env는 permissive |
| **암호화 없는 저장** | 대화 히스토리는 평문 SQLite |

---

## 15. 샌드박스 통합

### 지원 프로바이더

deepagents-cli는 **5개의 샌드박스 백엔드**를 지원한다:

| 프로바이더 | 유형 | 설명 |
|-----------|------|------|
| **AgentCore** | AWS 기반 | AWS 인프라 위의 격리 환경 |
| **Daytona** | 클라우드 | Daytona 개발 환경 |
| **LangSmith** | 클라우드 | LangSmith 통합 샌드박스 |
| **Modal** | 서버리스 | Modal.com 서버리스 컨테이너 |
| **Runloop** | 클라우드 | Runloop 격리 환경 |

### SandboxProvider 인터페이스

```python
class SandboxProvider(ABC):
    """모든 샌드박스 프로바이더의 추상 베이스 클래스"""

    # 동기 API
    def get_or_create(self, sandbox_id=None, **kwargs) -> SandboxBackend:
        """샌드박스를 가져오거나 새로 생성"""
        ...

    def delete(self, sandbox_id: str):
        """샌드박스 삭제"""
        ...

    # 비동기 API
    async def aget_or_create(self, sandbox_id=None, **kwargs) -> SandboxBackend:
        ...

    async def adelete(self, sandbox_id: str):
        ...
```

### 예외 계층

```python
class SandboxError(Exception):
    """샌드박스 기본 예외"""
    @property
    def original_exc(self) -> Exception | None:
        ...

class SandboxNotFoundError(SandboxError):
    """지정된 샌드박스를 찾을 수 없음"""
    ...
```

### 샌드박스 팩토리 (`sandbox_factory.py`)

컨텍스트 매니저로 샌드박스 생명주기를 관리한다:

```python
async with create_sandbox(sandbox_type="modal", sandbox_id="my-box") as sandbox:
    result = await sandbox.execute("ls -la")
    content = await sandbox.read_file("/app/main.py")
```

### 설정 스크립트 실행

샌드박스 초기화 시 설정 스크립트를 실행할 수 있다:

```python
# 환경 변수 확장: string.Template.safe_substitute()
# 인자 이스케이프: shlex.quote()
setup_script = "pip install ${PACKAGE_NAME} && python setup.py"
```

### 의존성 검증

각 프로바이더의 의존성은 **경량 검사**로 확인한다:

```python
# importlib.util.find_spec()으로 패키지 존재 확인
if not importlib.util.find_spec("modal"):
    raise SandboxError(
        "Modal sandbox requires 'modal' package. "
        "Install with: pip install modal"
    )
```

---

## 16. Non-Interactive 모드

### 헤드리스 실행

Non-Interactive 모드는 파이프라인, CI/CD, 스크립트에서 사용하기 위한 **헤드리스 실행 모드**다.

```bash
# 기본 사용
echo "이 코드의 버그를 분석해줘" | deepagents -n

# 셸 접근 허용
deepagents -n --shell-allow-list recommended -m "테스트를 실행하고 결과를 알려줘"

# 스트리밍 출력
deepagents -n --stream -m "README를 생성해줘"
```

### run_non_interactive() 진입점

```python
async def run_non_interactive(
    task: str,
    agent: CompiledGraph,
    stream: bool = False,
    ...
):
    """Non-Interactive 모드의 메인 함수"""
    ...
```

### HITL 승인 핸들링

Non-Interactive 모드에서도 HITL은 동작하지만, 자동화된 의사결정을 제공한다:

```
--shell-allow-list 미설정      → 셸 비활성화, 다른 도구 자동 승인
--shell-allow-list recommended → 안전 셸 허용, 다른 도구 자동 승인
--shell-allow-list all         → 모든 도구 자동 승인
```

#### 안전장치

- **최대 HITL 반복**: 50회 (무한 루프 방지)
- **Unicode/URL 안전성 검사**: 자동 실행
- **설정 기반 자동 결정**: allow-list에 따라 자동 승인/거부

### 출력 관리

| 채널 | 내용 |
|------|------|
| **stdout** | 에이전트 응답 텍스트 |
| **stderr** | 진단 정보 (quiet 모드 `-q` 시) |
| **스트리밍** | 실시간 토큰 출력 (`--stream`) |
| **버퍼링** | 완료 후 전체 출력 (기본) |

### StreamState 추적

```python
class StreamState:
    accumulated_text: str       # 누적된 텍스트
    tool_buffers: dict          # 도구 호출 버퍼
    interrupts: list            # HITL 인터럽트 목록
    decisions: list             # 자동 의사결정 이력
    token_usage: TokenStats     # 토큰 사용량 통계
```

---

## 17. 슬래시 명령어 & 커맨드 레지스트리

### 커맨드 레지스트리 (`command_registry.py`)

20개의 슬래시 명령어가 **5단계 바이패스 티어**로 분류된다. 각 티어는 명령어가 **어떤 상황에서 실행 가능한지**를 결정한다.

### 바이패스 티어

```
Tier 1: ALWAYS          ← 항상 실행 가능
Tier 2: CONNECTING       ← 서버 연결 중에도 실행 가능
Tier 3: IMMEDIATE_UI     ← 모달을 즉시 열 수 있음
Tier 4: SIDE_EFFECT_FREE ← 부작용 없는 즉시 실행
Tier 5: QUEUED           ← 앱 준비 후 큐에서 실행
```

### 명령어 목록

| Tier | 명령어 | 설명 |
|------|--------|------|
| **1: ALWAYS** | `/exit` | CLI 종료 |
| **2: CONNECTING** | `/model` | LLM 전환 |
| | `/threads` | 세션 브라우저 |
| | `/tokens` | 토큰 사용량 |
| **3: IMMEDIATE_UI** | `/skills` | 스킬 브라우저 |
| | `/remember` | 메모리 관리 |
| **4: SIDE_EFFECT_FREE** | `/help` | 도움말 시스템 |
| **5: QUEUED** | `/undo` | 마지막 변경 취소 |
| | `/clear` | 대화 초기화 |
| | `/shell-allow-list` | 셸 허용 목록 변경 |
| | `/reload-config` | 설정 다시 로드 |

### 동적 스킬 명령어

발견된 스킬은 자동으로 `/skill:<name>` 형식의 슬래시 명령어로 등록된다:

```python
def build_skill_commands(skills: dict) -> list[Command]:
    """발견된 스킬에서 /skill:<name> 명령어 생성"""
    commands = []
    for name, skill in skills.items():
        if not has_dedicated_command(name):  # 전용 명령어가 없는 스킬만
            commands.append(Command(
                name=f"skill:{name}",
                description=skill.description,
                tier=BypassTier.QUEUED
            ))
    return commands
```

---

## 18. 훅 & 이벤트 시스템

### 이벤트 훅 (`hooks.py`)

외부 프로세스를 이벤트에 반응하여 실행할 수 있는 시스템이다.

### 설정

`~/.deepagents/hooks.json`:

```json
{
  "hooks": [
    {
      "command": ["bash", "adapter.sh"],
      "events": ["session.start", "session.end"]
    },
    {
      "command": ["python", "logger.py"]
    }
  ]
}
```

- `events`를 지정하지 않으면 **모든 이벤트**에 반응한다.
- 각 훅은 별도의 서브프로세스로 실행된다.

### 지원 이벤트

| 이벤트 | 시점 |
|--------|------|
| `session.start` | CLI 세션 시작 |
| `session.end` | CLI 세션 종료 |
| 커스텀 이벤트 | 애플리케이션에서 정의 |

### 실행 특성

| 특성 | 설명 |
|------|------|
| **비동기 실행** | 백그라운드 스레드에서 실행 |
| **5초 타임아웃** | 명령당 최대 5초 |
| **병렬 실행** | ThreadPoolExecutor로 동시 실행 |
| **Fire-and-forget** | 강한 태스크 참조로 GC 방지 |
| **에러 격리** | 훅 실패가 CLI에 전파되지 않음 |

---

## 19. 테마 시스템

### 빌트인 테마

| 테마 | 스타일 | 기반 |
|------|--------|------|
| **Dark** | Tokyo Night 영감 | LangChain 블루 프라이머리 |
| **Light** | 따뜻한 뉴트럴 | 화이트 배경 |

### ThemeColors 데이터클래스

```python
@dataclass
class ThemeColors:
    """17개 시맨틱 색상 필드"""
    primary: str       # #RRGGBB 형식
    secondary: str
    success: str
    warning: str
    error: str
    surface: str
    background: str
    foreground: str
    muted: str
    accent: str
    # ... 7개 추가 필드

    def merged(self, overrides: dict) -> "ThemeColors":
        """커스텀 오버라이드를 적용한 새 인스턴스 반환"""
        ...
```

### 커스텀 테마 설정

`~/.deepagents/config.toml`:

```toml
[themes.tokyo-night]
label = "Tokyo Night"
dark = true
primary = "#7aa2f7"
secondary = "#bb9af7"
success = "#9ece6a"
warning = "#e0af68"
error = "#f7768e"
surface = "#1a1b26"
background = "#16161e"
foreground = "#c0caf5"
```

### 색상 해석 흐름

```
커스텀 테마  →  ThemeColors에서 직접 로드
빌트인 테마  →  Textual의 resolved CSS에서 파생
```

### ANSI 시맨틱 색상 (Non-Interactive)

Non-Interactive 모드에서는 Rich 콘솔 출력에 **ANSI 시맨틱 색상**을 사용한다:

```python
PRIMARY = "blue"
SUCCESS = "green"
WARNING = "yellow"
ERROR   = "red"
```

터미널 팔레트에 자동으로 적응한다.

---

## 20. 개발 & 테스트

### 테스트 조직

```
tests/
├── unit_tests/          # 빠른 격리 테스트
├── integration_tests/   # E2E 테스트 (30초 타임아웃)
└── README.md
```

### Makefile 타겟

```bash
make test              # 유닛 테스트 + 커버리지
make integration_test  # 통합 테스트
make test_watch        # Watch 모드
make lint              # Ruff 린트 + 타입 체크
make format            # 코드 포맷팅
make benchmark         # 벤치마크 테스트
make help              # 모든 타겟 표시
```

### 개발 환경 설정 (DEV.md)

#### Textual CSS Hot Reload

1. 개발 의존성 설치: `uv sync --group test` (textual-dev 포함)
2. 래퍼 스크립트 생성: `/tmp/dev_deepagents.py`
3. 터미널 1: `textual console` (Textual 콘솔)
4. 터미널 2: `textual run --dev /tmp/dev_deepagents.py` (핫 리로드)
5. `.tcss` 파일 편집 → 실시간 CSS 업데이트

### 주요 의존성

#### 코어 프레임워크

| 패키지 | 역할 |
|--------|------|
| `deepagents` | Deep Agents 코어 라이브러리 |
| `langchain` | LLM 추상화 레이어 |
| `langgraph` | 그래프 기반 에이전트 오케스트레이션 |
| `langgraph-cli` | LangGraph 개발 서버 |
| `textual` | TUI 프레임워크 |
| `rich` | 터미널 포맷팅 |
| `prompt-toolkit` | 입력 처리 |

#### LLM 프로바이더 (빌트인)

| 패키지 | 프로바이더 |
|--------|-----------|
| `langchain-anthropic` | Anthropic (Claude) |
| `langchain-google-genai` | Google (Gemini) |
| `langchain-openai` | OpenAI (GPT) |

#### LLM 프로바이더 (선택)

Groq, Ollama, LiteLLM, Baseten, Bedrock, Cohere, Deepseek, Perplexity 등 **13개 이상**의 추가 프로바이더를 `pip install`로 설치하여 사용할 수 있다.

#### 통합

| 패키지 | 역할 |
|--------|------|
| `tavily-python` | 웹 검색 |
| `langsmith` | 샌드박스 & 트레이싱 |
| `aiosqlite` | 비동기 SQLite |
| MCP 어댑터 | MCP 프로토콜 클라이언트 |

#### 개발 도구

| 도구 | 역할 |
|------|------|
| `ruff` | 린팅 & 포맷팅 |
| `pytest` | 테스트 (async/benchmark 지원) |
| `ty` | 타입 체크 |

---

## 21. 설정 파일 & 위치 총정리

### 사용자 레벨 (`~/.deepagents/`)

| 파일 | 역할 |
|------|------|
| `config.toml` | 모델/프로바이더/테마 설정, 스레드 표시 설정 |
| `.env` | 환경 변수 (API 키 등) |
| `.mcp.json` | 사용자 레벨 MCP 서버 설정 |
| `hooks.json` | 이벤트 훅 설정 |
| `*.db` | LangGraph 체크포인트 (대화 히스토리) |
| `history.jsonl` | 채팅 입력 히스토리 |

### 사용자 레벨 에이전트 (`~/.agents/`)

| 경로 | 역할 |
|------|------|
| `~/.agents/skills/` | 사용자 전역 스킬 |
| `~/.agents/agent/skills/` | 메인 에이전트 스킬 |
| `~/.agents/<name>/` | 에이전트별 스킬/설정 |

### 프로젝트 레벨

| 파일/경로 | 역할 |
|-----------|------|
| `.deepagents/config.toml` | 프로젝트 오버라이드 |
| `.deepagents/.mcp.json` | 프로젝트 MCP 설정 |
| `.deepagents/skills/` | 프로젝트 스킬 |
| `.deepagents/agents/` | 프로젝트 서브에이전트 |
| `.agents/` | 공유 프로젝트 스킬/에이전트 |
| `.env` | 프로젝트 환경 변수 |

### 설정 우선순위 다이어그램

```
CLI 인자  >  환경 변수  >  프로젝트 .env  >  사용자 ~/.deepagents/.env
    ▲            ▲              ▲                      ▲
  최고          높음            보통                    최저
```

---

## 22. 예제 워크플로우

### 워크플로우 1: 새 스킬 생성

```bash
# 방법 1: CLI 명령어로 기본 스킬 생성
deepagents skills create my-linter

# 방법 2: 스크립트로 리소스 포함 스킬 생성
python scripts/init_skill.py my-linter --path ~/.deepagents/agents

# 생성된 SKILL.md 편집
vim ~/.deepagents/agents/my-linter/SKILL.md

# 스킬 확인
deepagents skills info my-linter
deepagents skills list
```

### 워크플로우 2: 모델 런타임 오버라이드

```bash
# CLI 인자로 모델 & 파라미터 오버라이드
deepagents -m anthropic:claude-opus-4.1 -p '{"temperature": 0.7}'

# Interactive 모드에서 /model 명령으로 전환
# → /model 입력 → 퍼지 검색으로 모델 선택
```

### 워크플로우 3: Non-Interactive 태스크 실행

```bash
# 파이프 입력
echo "이 코드의 보안 취약점을 분석해줘" | deepagents -n

# 셸 접근 허용하여 실행
deepagents -n --shell-allow-list recommended -m "테스트를 실행하고 실패한 테스트를 수정해줘"

# 스트리밍 모드 (실시간 출력)
deepagents -n --stream -m "프로젝트 README를 생성해줘"

# Quiet 모드 (진단을 stderr로)
deepagents -n -q -m "빌드 스크립트를 최적화해줘" 2>/dev/null
```

### 워크플로우 4: 스레드 재개

```bash
# 가장 최근 스레드 재개
deepagents -r

# 특정 스레드 ID로 재개
deepagents -r abc-123-def-456

# 스레드 목록 확인
deepagents threads
deepagents threads --json
```

### 워크플로우 5: MCP 서버 연동

```bash
# 1. 프로젝트에 MCP 설정 파일 생성
cat > .mcp.json << 'EOF'
{
  "servers": {
    "my-tools": {
      "command": "node",
      "args": ["./mcp-server/index.js"],
      "type": "stdio"
    }
  }
}
EOF

# 2. deepagents 시작 → MCP 서버 자동 발견
deepagents -i

# 3. 첫 실행 시 신뢰 승인 프롬프트 표시
# → 승인하면 SHA-256 핑거프린트가 config.toml에 저장
```

### 워크플로우 6: 샌드박스 사용

```bash
# Modal 샌드박스에서 실행
deepagents -i --sandbox modal

# 특정 샌드박스 ID로 기존 환경 재사용
deepagents -i --sandbox modal --sandbox-id my-dev-box

# 설정 스크립트 포함
deepagents -i --sandbox daytona --sandbox-setup "pip install -r requirements.txt"
```

---

## 23. 핵심 설계 패턴

### 패턴 1: Lazy Loading 싱글턴

```python
class _SettingsProxy:
    """첫 접근 시까지 expensive 초기화를 지연"""
    _instance = None

    def __getattr__(self, name):
        if self._instance is None:
            self._instance = Settings()  # 여기서 .env 파싱, API 키 검사 등
        return getattr(self._instance, name)

settings = _SettingsProxy()
console = _ConsoleProxy()
```

**효과**: `deepagents help` 같은 가벼운 명령은 설정 로드 없이 ~65ms에 응답한다.

### 패턴 2: 3단계 캐싱

```
L1: LRU 메모리 캐시 (4,096 엔트리)
    └── 메시지 수, 초기 프롬프트
L2: 스레드 캐시 (16 조합, 신선도 토큰)
    └── 최근 스레드 목록
L3: 모델 프로파일 캐시 (스레드 안전 잠금)
    └── 프로바이더별 모델 목록
```

**신선도 보장**: 체크포인트 ID를 캐시 키에 포함하여, 상태 변경 시 자동 무효화.

### 패턴 3: 환경 변수 IPC

CLI와 서버 간 통신에 환경 변수를 사용한다:

```
CLI (main.py)                    서버 (server_graph.py)
     │                                │
     │  DA_SERVER_MODEL_NAME           │
     │  DA_SERVER_MODEL_PARAMS         │
     │  DA_SERVER_AUTO_APPROVE         │
     │  DA_SERVER_SHELL_ACCESS         │
     │  DA_SERVER_MCP_SETTINGS         │
     ├───────── env vars ─────────────▶│
     │                                │
     │                                │  ServerConfig.from_env()
     │                                │  → 환경 변수에서 설정 복원
```

**직렬화**: dict/list는 JSON, 문자열 리스트는 콤마 구분, 기본형은 직접 전달.

### 패턴 4: 미들웨어 합성

7개 미들웨어가 **체인 패턴**으로 합성되어, 기능을 모듈식으로 추가한다:

```
요청 → MW1 → MW2 → MW3 → MW4 → MW5 → MW6 → MW7 → LLM → 응답
         ↑                                              │
         └──────────── 응답 후처리 ◀────────────────────┘
```

**장점**:
- 각 미들웨어가 독립적으로 테스트 가능
- 새 기능 추가 시 미들웨어만 추가하면 됨
- 바이패스 티어로 UI 상태 전환 시 race condition 방지

### 패턴 5: 안전한 파일 연산 추적

```
FileOpTracker
    │
    ├── start()       # 경로, 액션 기록
    │
    ├── capture_before()  # 수정 전 내용 캡처
    │
    ├── complete()     # 수정 후 내용 캡처
    │
    ├── generate_diff()  # unified diff 생성 (설정 가능한 컨텍스트 라인)
    │
    └── approve()      # HITL 승인 후 실행
```

### 패턴 6: 적응형 폴링

서버 헬스 체크에 **환경에 따라 다른 폴링 간격**을 사용한다:

```python
poll_interval = 0.1 if is_local else 0.3  # 로컬: 100ms, 원격: 300ms
timeout = 60  # 최대 60초 대기
```

### 패턴 7: 메시지 가상화

대규모 대화의 렌더링 성능을 위해:

```
화면 밖 메시지 → 경량 플레이스홀더 (높이만 유지)
화면 안 메시지 → 전체 렌더링 (hydration)
스크롤 이벤트 → 보이는 메시지만 hydration
```

### 패턴 8: 타이핑 인식 인터럽트

```
도구 호출 승인 요청 발생
    │
    ├── 사용자 타이핑 중?
    │   ├── Yes → 큐에 적재, 2초 idle 대기
    │   └── No  → 즉시 다이얼로그 표시
    │
    └── idle 2초 경과 → 다이얼로그 표시
```

---

## 24. 정리 & 참고 자료

### 아키텍처 요약

deepagents-cli는 다음과 같은 **계층 구조**로 설계되어 있다:

```
┌─────────────────────────────────────────────────────┐
│                    사용자 인터페이스                    │
│  Textual TUI (Interactive) │ Headless (Non-Interactive)│
├─────────────────────────────────────────────────────┤
│                    미들웨어 체인                       │
│  Model │ Token │ AskUser │ Memory │ Skills │ Context  │
├─────────────────────────────────────────────────────┤
│                    에이전트 코어                       │
│  LangGraph Agent │ System Prompt │ HITL Interrupts    │
├─────────────────────────────────────────────────────┤
│                    도구 계층                           │
│  File Ops │ Shell │ Web │ MCP │ Subagents │ Skills   │
├─────────────────────────────────────────────────────┤
│                    인프라 계층                         │
│  LangGraph Server │ SQLite │ Sandbox │ Config │ Hooks │
└─────────────────────────────────────────────────────┘
```

### 핵심 수치 정리

| 항목 | 수치 |
|------|------|
| Python 소스 파일 | 47개 이상 |
| 위젯 파일 | 18개 |
| 미들웨어 | 7개 |
| 슬래시 명령어 | 20개 |
| MCP 전송 방식 | 3가지 (Stdio, SSE, HTTP) |
| 샌드박스 프로바이더 | 5개 |
| 스킬 발견 경로 | 7개 |
| LLM 프로바이더 | 15개 이상 |
| 바이패스 티어 | 5단계 |
| 활성 위협 | 10개 |
| 캐시 레이어 | 3단계 |

### 참고 자료

| 자료 | 위치 |
|------|------|
| 소스 코드 | [github.com/langchain-ai/deepagents/libs/cli](https://github.com/langchain-ai/deepagents/tree/main/libs/cli) |
| README | `libs/cli/README.md` |
| 위협 모델 | `libs/cli/THREAT_MODEL.md` |
| 변경 이력 | `libs/cli/CHANGELOG.md` |
| 개발 가이드 | `libs/cli/DEV.md` |
| 예제 스킬 | `libs/cli/examples/skills/` |
| LangGraph 문서 | [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph/) |
| Textual 문서 | [textual.textualize.io](https://textual.textualize.io/) |
| MCP 프로토콜 | [modelcontextprotocol.io](https://modelcontextprotocol.io/) |

---

> **문서 버전**: 2026-04-06 | **분석 대상**: deepagents-cli v0.0.34
