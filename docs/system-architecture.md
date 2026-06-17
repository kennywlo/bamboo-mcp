# Bamboo System Architecture

Bamboo is a tool-first MCP runtime. User interfaces are thin clients; routing,
tool execution, evidence handling, and LLM calls are handled server-side by the
Bamboo MCP server.

<img src="images/bamboo_system_architecture.png" alt="Bamboo system architecture" width="100%" />

```mermaid
flowchart TB
    user["User"]

    subgraph clients["Thin clients"]
        direction LR
        textual["Textual TUI"]
        streamlit["Streamlit UI"]
        inspector["MCP Inspector / curl"]
        external["External MCP host<br/>Claude Desktop / Cursor / custom client"]
    end

    subgraph transports["MCP transports"]
        direction LR
        stdio["stdio<br/>private server subprocess"]
        http["Streamable HTTP<br/>uvicorn ASGI /mcp"]
    end

    subgraph server["Bamboo MCP Server"]
        direction TB
        auth["HTTP auth + session state<br/>Bearer tokens, MCP session id"]
        core["core.py<br/>tools/list, tools/call, argument validation"]
        catalog["Tool catalog<br/>built-ins + plugin entry points"]

        subgraph orchestration["Server-side orchestration"]
            direction LR
            answer["bamboo_answer"]
            guard["Topic guard<br/>keyword check + optional fast LLM"]
            route["Routing<br/>fast-path rules or bamboo_plan"]
            exec["execute_plan"]
            synth["bamboo_llm_answer<br/>answer synthesis"]
        end

        health["bamboo_health<br/>bamboo_last_evidence"]
        tracing["Tracing / prompt logging<br/>local trace file or OpenSearch"]
    end

    subgraph tools["Tools and plugins"]
        direction LR
        rag["RAG tools<br/>panda_doc_search + panda_doc_bm25"]
        atlas["ATLAS / PanDA tools<br/>task, job, logs, queues, CRIC, Harvester"]
        epic["ePIC tools"]
        cgsim["AskCGSim tools<br/>docs + simulation queries"]
        custom["Additional Bamboo plugins<br/>via bamboo.tools entry points"]
    end

    subgraph data["Authoritative data sources"]
        direction LR
        chroma["ChromaDB collections<br/>atlas_docs, epic_docs, cgsim_docs"]
        bigpanda["BigPanDA HTTP API"]
        panda_mcp["External PanDA MCP server<br/>optional PANDA_MCP_BASE_URL"]
        dbs["DuckDB / SQLite stores<br/>CRIC, jobs, CGSim"]
        docs["Documentation corpora"]
        opensearch["CERN OpenSearch<br/>optional telemetry / prompt logs"]
    end

    subgraph llm["Pluggable LLM provider layer"]
        direction LR
        selector["LLM selector + client manager<br/>default / fast / reasoning profiles"]
        hosted["Hosted providers<br/>Mistral, OpenAI, Anthropic, Gemini"]
        compat["OpenAI-compatible endpoint<br/>vLLM, Ollama, LM Studio, etc."]
        tuned["Fine-tuned domain model<br/>for AskPanDA synthesis / planning"]
    end

    user --> textual
    user --> streamlit
    user --> inspector
    user --> external
    textual --> stdio
    streamlit --> stdio
    streamlit --> http
    inspector --> http
    external --> http

    stdio --> core
    http --> auth
    auth --> core

    core --> catalog
    core --> answer
    core --> health

    answer --> guard
    guard --> route
    route --> exec
    exec --> catalog
    catalog --> rag
    catalog --> atlas
    catalog --> epic
    catalog --> cgsim
    catalog --> custom
    exec --> synth

    synth --> selector
    guard -. optional classification .-> selector
    route -. planner fallback .-> selector
    selector --> hosted
    selector --> compat
    compat --> tuned

    rag --> chroma
    rag --> docs
    atlas --> bigpanda
    atlas --> panda_mcp
    atlas --> dbs
    cgsim --> dbs
    cgsim --> chroma
    epic --> bigpanda
    custom --> docs
    custom --> dbs

    tracing --> opensearch
```

## Component Roles

- **Thin clients** hold the user interaction and conversation history. They call
  Bamboo over MCP using stdio or Streamable HTTP.
- **Bamboo MCP Server** owns tool discovery, argument validation, routing,
  planning, execution, tracing, and LLM invocation.
- **`bamboo_answer`** is the primary server-side orchestrator used by Bamboo's
  own UIs.
- **Tools and plugins** fetch authoritative evidence. Plugin packages expose
  tools through the `bamboo.tools` entry point group.
- **Authoritative data sources** remain behind tools. The LLM does not query
  BigPanDA, ChromaDB, DuckDB, SQLite, OpenSearch, or upstream MCP servers
  directly.
- **The LLM provider layer** is swappable. A fine-tuned model fits behind an
  OpenAI-compatible endpoint such as vLLM or Ollama and is used for synthesis,
  optional guard classification, and planner fallback.

## Related Diagrams

- [MCP request sequence](mcp_sequence_diagram.mmd)
- [Tool-selection flow](mcp_tools_selection.mmd)
