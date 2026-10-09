---
title: "pi-agent 源码阅读"
date: 2026-10-10T01:00:00+08:00
draft: false
description: "pi-agent 源码阅读笔记"
slug: "ai-agent-learning-3-pi-agent"
tags: ["AI Agent", "Coding Agent", "pi-agent", "pi"]
categories: ["AI系统学习"]
ShowToc: true
TocOpen: true
---

# pi-agent 源码阅读

## 一、核心流程

入口函数位于 `packages/coding-agent/src/cli.ts` 处，其会调用 `main` 函数，`main` 函数的实现位于 `packages/coding-agent/src/main.ts:573` 。

### 1.0、特别注意

`packages` 这个目录下的不少包属于实验性功能。在初次阅读时可以先排除掉实验性功能外用不到的包，以精简阅读，然后再将实验性功能，如分布式的计算系统等功能，加回来再拓展性地阅读。

实验性功能相关的包如下表：

| 包                                        | 不开 experimental | 说明                                             |
| ----------------------------------------- | ----------------- | ------------------------------------------------ |
| `pi-tui`                                  | **必需**          | 交互模式的渲染基础                               |
| `pi-ai`                                   | **必需**          | 模型调用                                         |
| `pi-agent-core`                           | **必需**          | agent 循环                                       |
| `pi-telemetry`                            | **必需**          | `pi-ai` 的依赖                                   |
| `pi-codemode`                             | **必需**          | 内置 `codemode` 扩展（`extensions/index.ts:11`） |
| `pi-mcp`                                  | **必需**          | 内置 `mcp` 扩展（`extensions/index.ts:13`）      |
| `chord`                                   | 可移除*           | 仅 experimental 引用                             |
| `pi-client` / `pi-protocol` / `pi-server` | 可移除            | 仅 experimental + devDeps                        |
| `pi-durable` / `pi-env`                   | 可移除            | 仅 experimental                                  |
| `pi-evals`                                | 可移除            | 纯 dev 工具，不进产品                            |

### 1.1、核心包依赖图

```mermaid
flowchart TB
    subgraph L0["L0 基础库（零内部依赖）"]
        tui["pi-tui<br/>终端 UI 库"]
        mcp["pi-mcp<br/>MCP 客户端"]
        cm["pi-codemode<br/>沙箱 JS 执行"]
        tel["pi-telemetry<br/>遥测契约"]
    end

    subgraph L1["L1 模型能力"]
        ai["pi-ai<br/>统一多 provider LLM API"]
    end

    subgraph L2["L2 Agent 运行时"]
        core["pi-agent-core<br/>Agent 循环 + 工具调度"]
    end

    subgraph L3["L3 应用"]
        ca["pi-coding-agent<br/>CLI：TUI/print/RPC"]
    end

    ai --> tel
    core --> ai
    ca --> core
    ca --> ai
    ca --> tui
    ca --> mcp
    ca --> cm
```

## 1.2、coding-agent 内部模块分层

```mermaid
flowchart TB
    subgraph entry["入口 / 模式层"]
        cli["cli.ts → main.ts<br/>(脚手架/参数/装配)"]
        im["InteractiveMode<br/>(TUI 模式)"]
        pm["runPrintMode"]
        rpc["runRpcMode"]
    end

    subgraph assembly["组装层"]
        rt["agent-session-runtime.ts<br/>agent-session-services.ts"]
        sdk["sdk.ts<br/>createAgentSession: new Agent + AgentSession"]
    end

    subgraph session["会话核心层 (core/)"]
        as["AgentSession<br/>prompt/事件转发/持久化/压缩"]
        sm["SessionManager<br/>JSONL 会话文件"]
        mr["ModelRuntime<br/>模型选择/鉴权/目录"]
        rl["ResourceLoader<br/>扩展/技能/模板/主题"]
        st["SettingsManager / AuthStorage"]
    end

    subgraph capability["能力层"]
        tools["core/tools<br/>bash/read/edit/write/grep/find/ls"]
        ext["core/extensions<br/>内置: mcp/codemode/llama/tool-search"]
        comp["core/compaction<br/>上下文压缩"]
        sp["system-prompt / skills"]
    end

    cli --> im
    cli --> pm
    cli --> rpc
    im --> rt
    rt --> sdk
    sdk --> as
    as --> sm
    as --> mr
    as --> rl
    as --> st
    as --> comp
    as --> ext
    ext --> tools
    as --> sp
    im -.->|渲染| tui["pi-tui（包）"]
```

### 1.3、运行时调用时序（一次 prompt）

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant IM as InteractiveMode
    participant AS as AgentSession
    participant AG as Agent (pi-agent-core)
    participant AL as runLoop (agent-loop.ts)
    participant AI as pi-ai Provider
    participant T as 工具 (tools/扩展)

    U->>IM: 输入 + Enter (editor.onSubmit)
    IM->>AS: session.prompt(text)
    AS->>AS: 扩展钩子/技能与模板展开/模型校验
    AS->>AG: agent.prompt(messages)
    AG->>AL: runAgentLoop()
    loop 每轮
        AL->>AI: streamFn(model, convertToLlm(context))
        AI-->>AL: 流式 text/thinking/toolcall 增量
        AL-->>AS: AgentEvent (message_* / tool_execution_*)
        AS->>AS: 写入 SessionManager (JSONL)
        AS-->>IM: AgentSessionEvent
        IM-->>U: pi-tui 增量渲染
        alt 有 toolCall
            AL->>T: tool.execute(id, args)
            T-->>AL: toolResult → push 回 context
        end
    end
    AL-->>AS: agent_end
```

### 1.4、Agent 循环状态机（agent-loop.ts:163 内部）

```mermaid
flowchart TD
    S([prompt / continue]) --> A[streamAssistantResponse<br/>模型流式回复]
    A --> B{有 toolCall?}
    B -->|是| C[executeToolCalls<br/>串行或并行]
    C --> D[结果回填 context]
    D --> A
    B -->|否| E{steering 队列有消息?}
    E -->|有| A
    E -->|无| F{follow-up 队列有消息?}
    F -->|有| A
    F -->|无| G([agent_end])
```







## 二、分层解析

### 2.1、入口/模式层

pi 的运行实际有四种（`--mode` 选择）：interactive、text、json、rpc。其中 `runPrintMode` 一个函数同时承担 text 和 json 两种（print-mode.ts:2-7 的注释）。详细对比见下表：

| interactive | print (text/json) | rpc                   |                       |
| ----------- | ----------------- | --------------------- | --------------------- |
| 生命周期    | 常驻 TUI          | 一次性                | 常驻                  |
| 输入        | 终端键盘          | CLI 参数/管道         | stdin JSON 命令       |
| 输出        | TUI 渲染          | 最终文本 / JSONL 事件 | response + JSONL 事件 |
| 驱动者      | 人                | shell 脚本            | 外部程序              |

选择逻辑在 main.ts:112（`resolveAppMode`）：`--print`/`--mode` 显式指定；否则 stdin/stdout 是 TTY 就进 interactive，任一被重定向就退化为 print。三者共用完全相同的 `AgentSession` 内核，区别只是输入输出适配层。

### 2.2、AgentSession

```mermaid
flowchart TB
    subgraph sources["能力来源"]
        builtin["内置 ToolDefinition<br/>read/bash/edit/write/grep/find/ls"]
        ext["扩展注册 registerTool()<br/>第三方 + 内置(mcp/codemode/tool-search/llama)"]
        sdkT["SDK customTools"]
        skill["Skills (.pi/skills/*)<br/>Prompt templates"]
    end

    subgraph gov["AgentSession 工具/提示治理"]
        reg["_toolRegistry 全部注册工具"]
        loadout["_applyToolLoadout<br/>exposure: direct/codemode/deferred/hidden"]
        active["agent.state.tools<br/>= active 集(声明给模型)"]
        callable["_getCallableTools<br/>可被别的工具调用"]
        promptSections["buildSystemPromptSections<br/>preamble/tools/rules/docs/skills/cwd"]
    end

    subgraph loop["每轮请求"]
        agentLoop["pi-agent-core runLoop"]
        hooks["beforeToolCall / afterToolCall"]
        compact["_checkCompaction 压缩"]
    end

    builtin --> reg
    ext --> reg
    sdkT --> reg
    reg --> loadout --> active
    loadout --> callable
    active -->|"toolSnippets/toolGuidelines"| promptSections
    skill --> promptSections
    promptSections --> agentLoop
    agentLoop --> hooks
    hooks -->|"扩展事件"| ext
    agentLoop --> compact
    compact -->|"改写 context 后 continue"| agentLoop
```

