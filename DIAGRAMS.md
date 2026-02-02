# nanobot Mermaid 다이어그램 모음

> 개조 및 문서화를 위해 분리된 다이어그램들입니다.

---

## 1. 전체 시스템 아키텍처

```mermaid
flowchart TB
    subgraph External["외부 시스템"]
        LLM["LLM Provider<br/>OpenRouter/Anthropic/OpenAI"]
        Telegram["Telegram API"]
        WhatsApp["WhatsApp Web<br/>(Baileys)"]
        Web["Web Search<br/>(Brave)"]
    end

    subgraph Core["nanobot Core"]
        direction TB
        
        subgraph BusLayer["Message Bus Layer"]
            Bus["MessageBus<br/>async.Queue"]
            Inbound["Inbound Queue<br/>채널 → 에이전트"]
            Outbound["Outbound Queue<br/>에이전트 → 채널"]
        end
        
        subgraph AgentLayer["Agent Layer"]
            Loop["AgentLoop<br/>핵심 처리 엔진"]
            Context["ContextBuilder<br/>프롬프트 조립"]
            Tools["ToolRegistry<br/>도구 관리"]
            Subagent["SubagentManager<br/>백그라운드 작업"]
        end
        
        subgraph DataLayer["Data Layer"]
            Session["SessionManager<br/>대화 기록"]
            Memory["MemoryStore<br/>장기/단기 기억"]
            Skills["SkillsLoader<br/>스킬 관리"]
        end
        
        subgraph ServiceLayer["Service Layer"]
            Cron["CronService<br/>예약 작업"]
            Heartbeat["HeartbeatService<br/>주기적 깨어남"]
        end
        
        subgraph ChannelLayer["Channel Layer"]
            ChannelMgr["ChannelManager"]
            TG["TelegramChannel"]
            WA["WhatsAppChannel"]
        end
    end
    
    subgraph Bridge["Node.js Bridge"]
        WABridge["WhatsApp Bridge<br/>WebSocket 서버"]
    end

    Telegram <-->|HTTP Long Polling| TG
    WhatsApp <-->|WebSocket| WABridge
    WABridge <-->|WebSocket| WA
    
    TG -->|publish_inbound| Bus
    WA -->|publish_inbound| Bus
    Bus -->|consume_inbound| Loop
    Loop -->|publish_outbound| Bus
    Bus -->|dispatch| ChannelMgr
    ChannelMgr --> TG & WA
    
    Loop <-->|chat| LLM
    Loop <-->|use| Tools
    Loop <-->|build_messages| Context
    Loop <-->|get_history| Session
    Loop <-->|spawn| Subagent
    
    Context <-->|load| Memory
    Context <-->|load| Skills
    
    Cron -->|trigger| Loop
    Heartbeat -->|trigger| Loop
    
    Tools -->|web_search| Web
    Tools -->|exec| Shell["Shell/Filesystem"]
```

---

## 2. 메시지 처리 흐름 (Request Lifecycle)

```mermaid
sequenceDiagram
    actor User
    participant Channel as Telegram/WhatsApp
    participant Bus as MessageBus
    participant Agent as AgentLoop
    participant Context as ContextBuilder
    participant LLM as LLM Provider
    participant Tools as ToolRegistry
    participant Session as SessionManager

    User->>Channel: 메시지 전송
    Channel->>Bus: publish_inbound()
    Bus->>Agent: consume_inbound()
    
    Agent->>Session: get_or_create(session_key)
    Session-->>Agent: Session 객체
    
    Agent->>Context: build_messages(history, message)
    Context->>Context: load bootstrap files
    Context->>Context: load memory
    Context->>Context: load skills
    Context-->>Agent: messages[]
    
    loop Tool Iteration (max 20)
        Agent->>LLM: chat(messages, tools)
        LLM-->>Agent: LLMResponse
        
        alt has_tool_calls
            Agent->>Tools: execute(tool_name, params)
            Tools-->>Agent: result
            Agent->>Agent: add_tool_result(messages)
        else text response
            Agent->>Session: add_message(user, assistant)
            Session->>Session: save to disk
            Agent->>Bus: publish_outbound(response)
            Bus->>Channel: dispatch
            Channel->>User: 응답 전송
        end
    end
```

---

## 3. ReAct 패턴 상세

```mermaid
flowchart TD
    Start([사용자 입력]) --> Build[컨텍스트 빌드]
    Build --> LLM1[LLM 호출]
    LLM1 --> Decision{도구 호출?}
    
    Decision -->|Yes| Tool[도구 실행]
    Tool --> Result[결과를 메시지에 추가]
    Result --> LLM1
    
    Decision -->|No| Response[최종 응답 생성]
    Response --> Save[세션 저장]
    Save --> Send[사용자에게 전송]
    Send --> End([종료])
    
    style Decision fill:#f9f,stroke:#333
    style Tool fill:#bbf,stroke:#333
```

---

## 4. 데이터 흐름

```mermaid
flowchart LR
    subgraph Input["입력"]
        Msg["사용자 메시지"]
        Media["미디어 파일"]
        System["시스템 이벤트<br/>Cron/Heartbeat/Subagent"]
    end
    
    subgraph Processing["처리 파이프라인"]
        direction TB
        Parse["메시지 파싱"]
        Enrich["컨텍스트 강화"]
        Reason["LLM 추론"]
        Act["도구 실행"]
    end
    
    subgraph Storage["영구 저장소"]
        SessionFile["sessions/{key}.jsonl"]
        MemoryFile["memory/MEMORY.md"]
        DailyFile["memory/YYYY-MM-DD.md"]
        Config["~/.nanobot/config.json"]
    end
    
    subgraph Output["출력"]
        Response["텍스트 응답"]
        Action["외부 작업<br/>파일/명령/메시지"]
    end
    
    Msg --> Parse
    Media --> Parse
    System --> Parse
    
    Parse --> Enrich
    Enrich --> SessionFile
    Enrich --> MemoryFile
    
    Enrich --> Reason
    Reason --> Act
    Act --> Reason
    
    Reason --> Response
    Act --> Action
    Action --> DailyFile
```

---

## 5. 컨텍스트 빌드 구조

```mermaid
flowchart TB
    subgraph Context["ContextBuilder.build_system_prompt()"]
        direction TB
        
        Identity["1. Core Identity<br/>nanobot의 기본 정체성"]
        Bootstrap["2. Bootstrap Files<br/>AGENTS.md, SOUL.md, USER.md"]
        Memory["3. Memory<br/>Long-term + Daily"]
        Skills["4. Skills<br/>Always-loaded + Available"]
        
        Identity --> Bootstrap --> Memory --> Skills
    end
    
    subgraph Output["최종 프롬프트"]
        Prompt["""
        # nanobot 🐈
        You are nanobot...
        ---
        ## AGENTS.md
        ...
        ---
        ## Memory
        ...
        ---
        ## Skills
        ...
        """]
    end
    
    Skills --> Prompt
```

---

## 6. 메모리 시스템

```mermaid
flowchart TB
    subgraph MemorySystem["MemoryStore"]
        Daily["Daily Notes<br/>memory/YYYY-MM-DD.md"]
        LongTerm["Long-term Memory<br/>memory/MEMORY.md"]
    end
    
    subgraph Usage["사용 패턴"]
        Auto["자동 기록<br/>대화, 작업"]
        Manual["수동 기록<br/>중요한 사실"]
        Retrieve["조회<br/>최근 7일 + 장기"]
    end
    
    Auto --> Daily
    Manual --> LongTerm
    Daily --> Retrieve
    LongTerm --> Retrieve
```

---

## 7. 스킬 시스템

```mermaid
flowchart LR
    subgraph Sources["스킬 소스"]
        Builtin["내장 스킬<br/>nanobot/skills/"]
        Workspace["사용자 스킬<br/>~/.nanobot/workspace/skills/"]
    end
    
    subgraph Loading["로딩 전략"]
        Always["Always-loaded<br/>항상 컨텍스트에 포함"]
        Lazy["Lazy Loading<br/>목록만 제공, 필요시 로드"]
    end
    
    subgraph Check["요구사항 체크"]
        Binary["바이너리 존재?"]
        Env["환경변수 설정?"]
    end
    
    Builtin --> Loading
    Workspace --> Loading
    Loading --> Check
```

---

## 8. 도구 레지스트리

```mermaid
flowchart TB
    subgraph Registry["ToolRegistry"]
        Register["register(tool)"]
        Execute["execute(name, params)"]
        Schema["get_definitions() → OpenAI format"]
    end
    
    subgraph Tools["Built-in Tools"]
        Read["read_file"]
        Write["write_file"]
        Edit["edit_file"]
        List["list_dir"]
        Exec["exec<br/>shell command"]
        WebSearch["web_search"]
        WebFetch["web_fetch"]
        Message["message<br/>채널 메시지"]
        Spawn["spawn<br/>subagent"]
    end
    
    Register --> Tools
    Execute --> Tools
    Tools --> Schema
```

---

## 9. Subagent 통신 흐름

```mermaid
sequenceDiagram
    participant User
    participant Main as Main Agent
    participant Sub as Subagent
    participant Bus as MessageBus
    
    User->>Main: "코드 리뷰해줘 (큰 PR)"
    Main->>Sub: spawn("PR 분석")
    Main->>User: "백그라운드에서 분석 중..."
    
    Sub->>Sub: 독립적으로 작업 수행
    Sub->>Sub: GitHub API 호출
    Sub->>Sub: 코드 분석
    
    Sub->>Bus: publish_inbound(system channel)
    Bus->>Main: consume_inbound()
    Main->>User: "PR 분석 완료: 3개 이슈 발견..."
```

---

## 10. WhatsApp Bridge 아키텍처

```mermaid
flowchart LR
    subgraph Python["Python Side"]
        WA["WhatsAppChannel"]
    end
    
    subgraph Network["WebSocket"]
        WS["ws://localhost:3001"]
    end
    
    subgraph Node["Node.js Side"]
        Server["BridgeServer"]
        Client["WhatsAppClient<br/>(Baileys)"]
    end
    
    subgraph WhatsApp["WhatsApp Web"]
        Web["WhatsApp Web Protocol"]
    end
    
    WA <-->|websockets| WS
    WS <-->|ws| Server
    Server <-->|Baileys API| Client
    Client <-->|Binary Protocol| Web
```

---

## 11. Cron 서비스

```mermaid
flowchart TB
    subgraph Schedule["스케줄링 방식"]
        At["at<br/>특정 시간 1회"]
        Every["every<br/>주기적 실행"]
        Cron["cron<br/>Cron 표현식"]
    end
    
    subgraph Execution["실행 흐름"]
        Timer["타이머 설정"]
        Check["시간 체크"]
        Run["작업 실행"]
        Notify["결과 알림"]
    end
    
    Schedule --> Timer
    Timer --> Check
    Check -->|시간 도달| Run
    Run --> Notify
    Notify --> Timer
```

---

## 12. Heartbeat 서비스

```mermaid
flowchart TB
    File["HEARTBEAT.md"] --> Check{"내용 있음?"}
    Check -->|비어있음| Skip["건너뛰기"]
    Check -->|작업 있음| Trigger["에이전트 깨우기"]
    
    Trigger --> Response{"응답"}
    Response -->|HEARTBEAT_OK| Done["완료"]
    Response -->|작업 결과| Log["로그 기록"]
    
    style File fill:#f9f,stroke:#333
```

---

## 13. 세션 저장 구조

```mermaid
flowchart TB
    subgraph SessionDir["~/.nanobot/sessions/"]
        File1["telegram_123456789.jsonl"]
        File2["whatsapp_821012345678.jsonl"]
        File3["cli_default.jsonl"]
    end
    
    subgraph Format["파일 형식 (JSONL)"]
        Meta["{"_type":"metadata",...}"]
        Msg1["{"role":"user",...}"]
        Msg2["{"role":"assistant",...}"]
    end
    
    File1 --> Format
```

---

## 14. 클래스 의존성 다이어그램

```mermaid
classDiagram
    class AgentLoop {
        +MessageBus bus
        +LLMProvider provider
        +ContextBuilder context
        +SessionManager sessions
        +ToolRegistry tools
        +SubagentManager subagents
        +run()
        +_process_message()
    }
    
    class MessageBus {
        +Queue inbound
        +Queue outbound
        +publish_inbound()
        +consume_inbound()
        +publish_outbound()
    }
    
    class LLMProvider {
        <<abstract>>
        +chat()
        +get_default_model()
    }
    
    class LiteLLMProvider {
        +chat()
    }
    
    class ContextBuilder {
        +MemoryStore memory
        +SkillsLoader skills
        +build_system_prompt()
        +build_messages()
    }
    
    class ToolRegistry {
        +register()
        +execute()
        +get_definitions()
    }
    
    class Tool {
        <<abstract>>
        +name
        +description
        +parameters
        +execute()
    }
    
    class SessionManager {
        +get_or_create()
        +save()
    }
    
    class SubagentManager {
        +spawn()
        +_run_subagent()
    }
    
    AgentLoop --> MessageBus
    AgentLoop --> LLMProvider
    AgentLoop --> ContextBuilder
    AgentLoop --> ToolRegistry
    AgentLoop --> SessionManager
    AgentLoop --> SubagentManager
    
    LLMProvider <|-- LiteLLMProvider
    ToolRegistry --> Tool
    ContextBuilder --> MemoryStore
    ContextBuilder --> SkillsLoader
```

---

## 15. 모듈 의존성 그래프

```mermaid
flowchart TB
    subgraph CLI["CLI Layer"]
        Commands["cli/commands.py"]
    end
    
    subgraph Core["Core Layer"]
        Agent["agent/"]
        Bus["bus/"]
        Channels["channels/"]
    end
    
    subgraph Support["Support Layer"]
        Providers["providers/"]
        Session["session/"]
        Config["config/"]
        Utils["utils/"]
    end
    
    subgraph Services["Service Layer"]
        Cron["cron/"]
        Heartbeat["heartbeat/"]
    end
    
    Commands --> Agent
    Commands --> Channels
    Commands --> Services
    
    Agent --> Bus
    Agent --> Providers
    Agent --> Session
    Agent --> Config
    
    Channels --> Bus
    Channels --> Config
    
    Services --> Agent
    Services --> Config
```

---

## 16. 상태 머신: 에이전트 처리 상태

```mermaid
stateDiagram-v2
    [*] --> Idle: 시작
    Idle --> Processing: 메시지 수신
    
    Processing --> LLMCall: 컨텍스트 준비
    LLMCall --> ToolExecution: 도구 호출
    ToolExecution --> LLMCall: 결과 반영
    
    LLMCall --> Responding: 텍스트 응답
    Responding --> Saving: 세션 저장
    Saving --> Idle: 대기
    
    Processing --> Error: 예외 발생
    ToolExecution --> Error: 도구 오류
    Error --> Responding: 오류 응답
```

---

## 17. 배포 아키텍처

```mermaid
flowchart TB
    subgraph Local["로컬 개발"]
        DevCLI["nanobot agent -m \"...\""]
        DevGateway["nanobot gateway"]
    end
    
    subgraph Server["서버 배포"]
        Systemd["systemd service"]
        Docker["Docker container"]
        PM2["PM2 process manager"]
    end
    
    subgraph External["외부 연동"]
        OpenRouter["OpenRouter API"]
        TelegramAPI["Telegram Bot API"]
        WhatsAppWeb["WhatsApp Web"]
    end
    
    DevCLI --> OpenRouter
    DevGateway --> TelegramAPI
    DevGateway --> WhatsAppWeb
    
    Systemd --> OpenRouter
    Docker --> TelegramAPI
    PM2 --> WhatsAppWeb
```

---

> 💡 **팁**: Mermaid 다이어그램은 GitHub, Notion, Obsidian 등에서 직접 렌더링됩니다.
